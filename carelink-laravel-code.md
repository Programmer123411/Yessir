# CareLink Clinic Management System - Laravel Backend

Laravel 11 + Sanctum (token auth) + MySQL. Same modules as the Node version: Dashboard, Information Management (Patients, Appointments, Medical Records), Reports, User Management, Activity Logs (audit trail).

## 1. Setup

```
composer create-project laravel/laravel carelink-api
cd carelink-api
php artisan install:api
```

Set your database in `.env` (DB_CONNECTION=mysql, DB_DATABASE=carelink, DB_USERNAME, DB_PASSWORD).

Create the files:

```
php artisan make:model Patient
php artisan make:model Appointment
php artisan make:model MedicalRecord
php artisan make:model ActivityLog
php artisan make:middleware RoleMiddleware
php artisan make:controller AuthController
php artisan make:controller UserController
php artisan make:controller PatientController
php artisan make:controller AppointmentController
php artisan make:controller MedicalRecordController
php artisan make:controller DashboardController
php artisan make:controller ReportController
php artisan make:controller ActivityLogController
```

Create `app/Services/ActivityLogger.php` by hand, paste every file below, then run:

```
php artisan migrate
php artisan db:seed
php artisan serve
```

Test with Postman: send the headers `Accept: application/json` and, after login, `Authorization: Bearer <token>`.

Default accounts (change them right away): admin@carelink.com / Admin1234, doctor@carelink.com / Doctor1234, nurse@carelink.com / Nurse1234, staff@carelink.com / Staff1234.

---

## 2. bootstrap/app.php

Add the middleware alias and the JSON error rendering inside the existing file:

```php
<?php

use Illuminate\Foundation\Application;
use Illuminate\Foundation\Configuration\Exceptions;
use Illuminate\Foundation\Configuration\Middleware;

return Application::configure(basePath: dirname(__DIR__))
    ->withRouting(
        web: __DIR__.'/../routes/web.php',
        api: __DIR__.'/../routes/api.php',
        commands: __DIR__.'/../routes/console.php',
        health: '/up',
    )
    ->withMiddleware(function (Middleware $middleware) {
        $middleware->alias([
            'role' => \App\Http\Middleware\RoleMiddleware::class,
        ]);
    })
    ->withExceptions(function (Exceptions $exceptions) {
        $exceptions->shouldRenderJsonWhen(
            fn ($request, $e) => $request->is('api/*') || $request->expectsJson()
        );
    })->create();
```

---

## 3. Migrations (database/migrations)

Name them in this order so foreign keys work.

### 2024_01_01_000001_add_role_to_users_table.php

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration {
    public function up(): void
    {
        Schema::table('users', function (Blueprint $table) {
            $table->enum('role', ['admin', 'doctor', 'nurse', 'staff'])->default('staff');
            $table->enum('status', ['active', 'inactive'])->default('active');
            $table->timestamp('last_login')->nullable();
        });
    }

    public function down(): void
    {
        Schema::table('users', function (Blueprint $table) {
            $table->dropColumn(['role', 'status', 'last_login']);
        });
    }
};
```

### 2024_01_01_000002_create_patients_table.php

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration {
    public function up(): void
    {
        Schema::create('patients', function (Blueprint $table) {
            $table->id();
            $table->string('patient_id')->nullable()->unique();
            $table->string('first_name');
            $table->string('last_name');
            $table->date('date_of_birth');
            $table->enum('gender', ['male', 'female', 'other']);
            $table->string('phone');
            $table->string('email')->nullable();
            $table->string('address')->nullable();
            $table->string('blood_type', 3)->nullable();
            $table->json('allergies')->nullable();
            $table->json('emergency_contact')->nullable();
            $table->foreignId('created_by')->nullable()->constrained('users')->nullOnDelete();
            $table->timestamps();
            $table->softDeletes();
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('patients');
    }
};
```

### 2024_01_01_000003_create_appointments_table.php

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration {
    public function up(): void
    {
        Schema::create('appointments', function (Blueprint $table) {
            $table->id();
            $table->foreignId('patient_id')->constrained('patients');
            $table->foreignId('doctor_id')->constrained('users');
            $table->date('date');
            $table->string('time', 5); // HH:mm
            $table->string('reason')->nullable();
            $table->enum('status', ['scheduled', 'completed', 'cancelled', 'no-show'])->default('scheduled');
            $table->text('notes')->nullable();
            $table->foreignId('created_by')->nullable()->constrained('users')->nullOnDelete();
            $table->timestamps();
            $table->index(['doctor_id', 'date', 'time']);
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('appointments');
    }
};
```

### 2024_01_01_000004_create_medical_records_table.php

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration {
    public function up(): void
    {
        Schema::create('medical_records', function (Blueprint $table) {
            $table->id();
            $table->foreignId('patient_id')->constrained('patients');
            $table->foreignId('doctor_id')->constrained('users');
            $table->foreignId('appointment_id')->nullable()->constrained('appointments')->nullOnDelete();
            $table->date('visit_date');
            $table->json('vitals')->nullable();
            $table->text('symptoms')->nullable();
            $table->text('diagnosis');
            $table->text('treatment')->nullable();
            $table->json('prescriptions')->nullable();
            $table->text('notes')->nullable();
            $table->timestamps();
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('medical_records');
    }
};
```

### 2024_01_01_000005_create_activity_logs_table.php

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration {
    public function up(): void
    {
        Schema::create('activity_logs', function (Blueprint $table) {
            $table->id();
            $table->foreignId('user_id')->nullable()->constrained('users')->nullOnDelete();
            $table->string('user_name')->default('Anonymous');
            $table->string('role')->nullable();
            $table->string('action'); // CREATE READ UPDATE DELETE LOGIN LOGOUT LOGIN_FAILED
            $table->string('module');
            $table->text('description')->nullable();
            $table->string('target_id')->nullable();
            $table->json('before')->nullable();
            $table->json('after')->nullable();
            $table->string('ip_address', 45)->nullable();
            $table->string('user_agent')->nullable();
            $table->timestamp('created_at')->useCurrent()->index();
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('activity_logs');
    }
};
```

---

## 4. Models (app/Models)

### User.php

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Foundation\Auth\User as Authenticatable;
use Illuminate\Notifications\Notifiable;
use Laravel\Sanctum\HasApiTokens;

class User extends Authenticatable
{
    use HasApiTokens, HasFactory, Notifiable;

    protected $fillable = ['name', 'email', 'password', 'role', 'status', 'last_login'];

    protected $hidden = ['password', 'remember_token'];

    protected function casts(): array
    {
        return [
            'password' => 'hashed',
            'last_login' => 'datetime',
        ];
    }
}
```

### Patient.php

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\SoftDeletes;

class Patient extends Model
{
    use SoftDeletes;

    protected $fillable = [
        'first_name', 'last_name', 'date_of_birth', 'gender', 'phone', 'email',
        'address', 'blood_type', 'allergies', 'emergency_contact', 'created_by',
    ];

    protected $casts = [
        'date_of_birth' => 'date:Y-m-d',
        'allergies' => 'array',
        'emergency_contact' => 'array',
    ];

    // Auto-generate readable ID: P-000001
    protected static function booted(): void
    {
        static::created(function (Patient $patient) {
            $patient->patient_id = sprintf('P-%06d', $patient->id);
            $patient->saveQuietly();
        });
    }
}
```

### Appointment.php

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;

class Appointment extends Model
{
    protected $fillable = [
        'patient_id', 'doctor_id', 'date', 'time', 'reason', 'status', 'notes', 'created_by',
    ];

    protected $casts = ['date' => 'date:Y-m-d'];

    public function patient()
    {
        return $this->belongsTo(Patient::class)->withTrashed();
    }

    public function doctor()
    {
        return $this->belongsTo(User::class, 'doctor_id');
    }
}
```

### MedicalRecord.php

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;

class MedicalRecord extends Model
{
    protected $fillable = [
        'patient_id', 'doctor_id', 'appointment_id', 'visit_date', 'vitals',
        'symptoms', 'diagnosis', 'treatment', 'prescriptions', 'notes',
    ];

    protected $casts = [
        'visit_date' => 'date:Y-m-d',
        'vitals' => 'array',
        'prescriptions' => 'array',
    ];

    public function patient()
    {
        return $this->belongsTo(Patient::class)->withTrashed();
    }

    public function doctor()
    {
        return $this->belongsTo(User::class, 'doctor_id');
    }
}
```

### ActivityLog.php

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;

class ActivityLog extends Model
{
    const UPDATED_AT = null; // append-only: no updated_at column

    protected $fillable = [
        'user_id', 'user_name', 'role', 'action', 'module', 'description',
        'target_id', 'before', 'after', 'ip_address', 'user_agent',
    ];

    protected $casts = ['before' => 'array', 'after' => 'array'];

    // Audit logs are immutable: block edits and deletes at the model level
    protected static function booted(): void
    {
        static::updating(fn () => throw new \RuntimeException('Activity logs are immutable'));
        static::deleting(fn () => throw new \RuntimeException('Activity logs are immutable'));
    }
}
```

---

## 5. Audit trail service

### app/Services/ActivityLogger.php

```php
<?php

namespace App\Services;

use App\Models\ActivityLog;
use App\Models\User;

class ActivityLogger
{
    /**
     * Writes one audit entry. Never throws: logging must not break a request.
     * $action: CREATE | READ | UPDATE | DELETE | LOGIN | LOGOUT | LOGIN_FAILED
     */
    public static function log(
        string $action,
        string $module,
        string $description,
        $targetId = null,
        ?array $before = null,
        ?array $after = null,
        ?User $user = null
    ): void {
        try {
            $user = $user ?? auth()->user();

            ActivityLog::create([
                'user_id' => $user?->id,
                'user_name' => $user?->name ?? 'Anonymous',
                'role' => $user?->role,
                'action' => $action,
                'module' => $module,
                'description' => $description,
                'target_id' => $targetId ? (string) $targetId : null,
                'before' => $before,
                'after' => $after,
                'ip_address' => request()->ip(),
                'user_agent' => substr((string) request()->userAgent(), 0, 255),
            ]);
        } catch (\Throwable $e) {
            report($e);
        }
    }
}
```

---

## 6. Role middleware

### app/Http/Middleware/RoleMiddleware.php

```php
<?php

namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;

class RoleMiddleware
{
    // Usage: ->middleware('role:admin,doctor')
    public function handle(Request $request, Closure $next, string ...$roles)
    {
        $user = $request->user();

        if (! $user || ! in_array($user->role, $roles, true)) {
            return response()->json([
                'success' => false,
                'message' => 'You do not have permission to perform this action.',
            ], 403);
        }

        return $next($request);
    }
}
```

---

## 7. Controllers (app/Http/Controllers)

### AuthController.php

```php
<?php

namespace App\Http\Controllers;

use App\Models\User;
use App\Services\ActivityLogger;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Hash;
use Illuminate\Validation\Rules\Password;

class AuthController extends Controller
{
    // POST /api/auth/login
    public function login(Request $request)
    {
        $data = $request->validate([
            'email' => 'required|email',
            'password' => 'required|string',
        ]);

        $user = User::where('email', $data['email'])->first();

        if (! $user || ! Hash::check($data['password'], $user->password)) {
            ActivityLogger::log('LOGIN_FAILED', 'Auth', "Failed login attempt for {$data['email']}");
            return response()->json(['success' => false, 'message' => 'Invalid email or password'], 401);
        }

        if ($user->status !== 'active') {
            return response()->json(['success' => false, 'message' => 'Account is deactivated'], 403);
        }

        $user->update(['last_login' => now()]);
        ActivityLogger::log('LOGIN', 'Auth', "{$user->name} logged in", $user->id, null, null, $user);

        return response()->json([
            'success' => true,
            'token' => $user->createToken('carelink')->plainTextToken,
            'user' => $user->only(['id', 'name', 'email', 'role']),
        ]);
    }

    // GET /api/auth/me
    public function me(Request $request)
    {
        return response()->json(['success' => true, 'user' => $request->user()]);
    }

    // POST /api/auth/logout
    public function logout(Request $request)
    {
        ActivityLogger::log('LOGOUT', 'Auth', "{$request->user()->name} logged out");
        $request->user()->currentAccessToken()->delete();

        return response()->json(['success' => true, 'message' => 'Logged out']);
    }

    // PUT /api/auth/change-password
    public function changePassword(Request $request)
    {
        $data = $request->validate([
            'current_password' => 'required|string',
            'new_password' => ['required', Password::min(8)->mixedCase()->numbers()],
        ]);

        $user = $request->user();

        if (! Hash::check($data['current_password'], $user->password)) {
            return response()->json(['success' => false, 'message' => 'Current password is incorrect'], 401);
        }

        $user->update(['password' => $data['new_password']]);
        ActivityLogger::log('UPDATE', 'Auth', "{$user->name} changed their password", $user->id);

        return response()->json(['success' => true, 'message' => 'Password updated']);
    }
}
```

### UserController.php

```php
<?php

namespace App\Http\Controllers;

use App\Models\User;
use App\Services\ActivityLogger;
use Illuminate\Http\Request;
use Illuminate\Validation\Rule;
use Illuminate\Validation\Rules\Password;

class UserController extends Controller
{
    private const SNAPSHOT = ['name', 'email', 'role', 'status'];

    // GET /api/users?search=&role=&status=&limit=
    public function index(Request $request)
    {
        $users = User::query()
            ->when($request->role, fn ($q, $v) => $q->where('role', $v))
            ->when($request->status, fn ($q, $v) => $q->where('status', $v))
            ->when($request->search, fn ($q, $s) => $q->where(
                fn ($w) => $w->where('name', 'like', "%$s%")->orWhere('email', 'like', "%$s%")
            ))
            ->latest()
            ->paginate($request->integer('limit', 10));

        return response()->json(['success' => true] + $users->toArray());
    }

    public function show(User $user)
    {
        return response()->json(['success' => true, 'data' => $user]);
    }

    public function store(Request $request)
    {
        $data = $request->validate([
            'name' => 'required|string|min:2|max:80',
            'email' => 'required|email|unique:users,email',
            'password' => ['required', Password::min(8)->mixedCase()->numbers()],
            'role' => 'required|in:admin,doctor,nurse,staff',
        ]);

        $user = User::create($data);

        ActivityLogger::log('CREATE', 'Users', "Created user {$user->name} ({$user->role})",
            $user->id, null, $user->only(self::SNAPSHOT));

        return response()->json(['success' => true, 'data' => $user], 201);
    }

    public function update(Request $request, User $user)
    {
        $data = $request->validate([
            'name' => 'sometimes|string|min:2|max:80',
            'email' => ['sometimes', 'email', Rule::unique('users', 'email')->ignore($user->id)],
            'password' => ['sometimes', Password::min(8)->mixedCase()->numbers()],
            'role' => 'sometimes|in:admin,doctor,nurse,staff',
            'status' => 'sometimes|in:active,inactive',
        ]);

        $demotingSelf = ($data['status'] ?? null) === 'inactive'
            || (isset($data['role']) && $data['role'] !== 'admin');

        if ($user->id === $request->user()->id && $demotingSelf) {
            return response()->json([
                'success' => false,
                'message' => 'You cannot deactivate or demote your own account',
            ], 400);
        }

        $before = $user->only(self::SNAPSHOT);
        $user->update($data);

        // Deactivated users lose access immediately
        if (($data['status'] ?? null) === 'inactive') {
            $user->tokens()->delete();
        }

        ActivityLogger::log('UPDATE', 'Users', "Updated user {$user->name}",
            $user->id, $before, $user->only(self::SNAPSHOT));

        return response()->json(['success' => true, 'data' => $user]);
    }

    public function destroy(Request $request, User $user)
    {
        if ($user->id === $request->user()->id) {
            return response()->json(['success' => false, 'message' => 'You cannot delete your own account'], 400);
        }

        $before = $user->only(self::SNAPSHOT);
        $user->tokens()->delete();
        $user->delete();

        ActivityLogger::log('DELETE', 'Users', "Deleted user {$before['name']}", $user->id, $before);

        return response()->json(['success' => true, 'message' => 'User deleted']);
    }
}
```

### PatientController.php

```php
<?php

namespace App\Http\Controllers;

use App\Models\Patient;
use App\Services\ActivityLogger;
use Illuminate\Http\Request;

class PatientController extends Controller
{
    private function rules(bool $create): array
    {
        $r = $create ? 'required' : 'sometimes';

        return [
            'first_name' => "$r|string|max:100",
            'last_name' => "$r|string|max:100",
            'date_of_birth' => "$r|date|before_or_equal:today",
            'gender' => "$r|in:male,female,other",
            'phone' => [$r, 'regex:/^[0-9+\-\s()]{7,20}$/'],
            'email' => 'nullable|email',
            'address' => 'nullable|string|max:255',
            'blood_type' => 'nullable|in:A+,A-,B+,B-,AB+,AB-,O+,O-',
            'allergies' => 'nullable|array',
            'allergies.*' => 'string',
            'emergency_contact' => 'nullable|array',
            'emergency_contact.name' => 'nullable|string',
            'emergency_contact.phone' => 'nullable|string',
            'emergency_contact.relationship' => 'nullable|string',
        ];
    }

    // GET /api/patients?search=&gender=&limit=
    public function index(Request $request)
    {
        $patients = Patient::query()
            ->when($request->gender, fn ($q, $g) => $q->where('gender', $g))
            ->when($request->search, fn ($q, $s) => $q->where(
                fn ($w) => $w->where('first_name', 'like', "%$s%")
                    ->orWhere('last_name', 'like', "%$s%")
                    ->orWhere('patient_id', 'like', "%$s%")
                    ->orWhere('phone', 'like', "%$s%")
            ))
            ->latest()
            ->paginate($request->integer('limit', 10));

        return response()->json(['success' => true] + $patients->toArray());
    }

    public function show(Patient $patient)
    {
        ActivityLogger::log('READ', 'Patients', "Viewed patient {$patient->patient_id}", $patient->id);

        return response()->json(['success' => true, 'data' => $patient]);
    }

    public function store(Request $request)
    {
        $data = $request->validate($this->rules(true));

        $patient = Patient::create($data + ['created_by' => $request->user()->id]);
        $patient->refresh();

        ActivityLogger::log('CREATE', 'Patients',
            "Registered patient {$patient->patient_id} - {$patient->first_name} {$patient->last_name}",
            $patient->id, null, $patient->toArray());

        return response()->json(['success' => true, 'data' => $patient], 201);
    }

    public function update(Request $request, Patient $patient)
    {
        $data = $request->validate($this->rules(false));

        $before = $patient->toArray();
        $patient->update($data);

        ActivityLogger::log('UPDATE', 'Patients', "Updated patient {$patient->patient_id}",
            $patient->id, $before, $patient->fresh()->toArray());

        return response()->json(['success' => true, 'data' => $patient]);
    }

    // Soft delete: medical history stays intact
    public function destroy(Patient $patient)
    {
        $patient->delete();

        ActivityLogger::log('DELETE', 'Patients', "Archived patient {$patient->patient_id}", $patient->id);

        return response()->json(['success' => true, 'message' => 'Patient archived']);
    }
}
```

### AppointmentController.php

```php
<?php

namespace App\Http\Controllers;

use App\Models\Appointment;
use App\Services\ActivityLogger;
use Illuminate\Http\Request;
use Illuminate\Validation\Rule;

class AppointmentController extends Controller
{
    private function rules(bool $create): array
    {
        $r = $create ? 'required' : 'sometimes';

        return [
            'patient_id' => [$r, Rule::exists('patients', 'id')->whereNull('deleted_at')],
            'doctor_id' => [$r, Rule::exists('users', 'id')->where('role', 'doctor')->where('status', 'active')],
            'date' => "$r|date",
            'time' => [$r, 'regex:/^([01]\d|2[0-3]):[0-5]\d$/'],
            'status' => 'sometimes|in:scheduled,completed,cancelled,no-show',
            'reason' => 'nullable|string|max:300',
            'notes' => 'nullable|string',
        ];
    }

    // Prevent double-booking the same doctor at the same slot
    private function clashes($doctorId, $date, $time, ?int $ignoreId = null): bool
    {
        return Appointment::where('doctor_id', $doctorId)
            ->whereDate('date', $date)
            ->where('time', $time)
            ->where('status', 'scheduled')
            ->when($ignoreId, fn ($q) => $q->where('id', '!=', $ignoreId))
            ->exists();
    }

    private function with(): array
    {
        return ['patient:id,patient_id,first_name,last_name,phone', 'doctor:id,name,role'];
    }

    // GET /api/appointments?status=&doctor_id=&date=&limit=
    public function index(Request $request)
    {
        $appointments = Appointment::with($this->with())
            ->when($request->status, fn ($q, $v) => $q->where('status', $v))
            ->when($request->doctor_id, fn ($q, $v) => $q->where('doctor_id', $v))
            ->when($request->date, fn ($q, $v) => $q->whereDate('date', $v))
            ->orderBy('date')->orderBy('time')
            ->paginate($request->integer('limit', 10));

        return response()->json(['success' => true] + $appointments->toArray());
    }

    public function show(Appointment $appointment)
    {
        return response()->json(['success' => true, 'data' => $appointment->load($this->with())]);
    }

    public function store(Request $request)
    {
        $data = $request->validate($this->rules(true));

        if ($this->clashes($data['doctor_id'], $data['date'], $data['time'])) {
            return response()->json([
                'success' => false,
                'message' => 'Doctor already has an appointment at this time',
            ], 409);
        }

        $appointment = Appointment::create($data + ['created_by' => $request->user()->id]);

        ActivityLogger::log('CREATE', 'Appointments',
            "Scheduled appointment on {$data['date']} {$data['time']}",
            $appointment->id, null, $appointment->toArray());

        return response()->json(['success' => true, 'data' => $appointment], 201);
    }

    public function update(Request $request, Appointment $appointment)
    {
        $data = $request->validate($this->rules(false));

        $final = array_merge($appointment->only(['doctor_id', 'date', 'time', 'status']), $data);

        if ($final['status'] === 'scheduled'
            && $this->clashes($final['doctor_id'], $final['date'], $final['time'], $appointment->id)) {
            return response()->json([
                'success' => false,
                'message' => 'Doctor already has an appointment at this time',
            ], 409);
        }

        $before = $appointment->toArray();
        $appointment->update($data);

        ActivityLogger::log('UPDATE', 'Appointments', "Updated appointment (status: {$appointment->status})",
            $appointment->id, $before, $appointment->fresh()->toArray());

        return response()->json(['success' => true, 'data' => $appointment]);
    }

    public function destroy(Appointment $appointment)
    {
        $before = $appointment->toArray();
        $appointment->delete();

        ActivityLogger::log('DELETE', 'Appointments', 'Deleted appointment', $appointment->id, $before);

        return response()->json(['success' => true, 'message' => 'Appointment deleted']);
    }
}
```

### MedicalRecordController.php

```php
<?php

namespace App\Http\Controllers;

use App\Models\MedicalRecord;
use App\Services\ActivityLogger;
use Illuminate\Http\Request;
use Illuminate\Validation\Rule;

class MedicalRecordController extends Controller
{
    private function rules(bool $create): array
    {
        $r = $create ? 'required' : 'sometimes';

        return [
            'patient_id' => [$r, Rule::exists('patients', 'id')->whereNull('deleted_at')],
            'doctor_id' => 'sometimes|exists:users,id',
            'appointment_id' => 'nullable|exists:appointments,id',
            'visit_date' => 'nullable|date',
            'vitals' => 'nullable|array',
            'vitals.blood_pressure' => 'nullable|string|max:20',
            'vitals.heart_rate' => 'nullable|integer|between:20,250',
            'vitals.temperature' => 'nullable|numeric|between:30,45',
            'vitals.weight' => 'nullable|numeric|between:0.5,500',
            'vitals.height' => 'nullable|numeric|between:20,280',
            'symptoms' => 'nullable|string',
            'diagnosis' => "$r|string",
            'treatment' => 'nullable|string',
            'prescriptions' => 'nullable|array',
            'prescriptions.*.medicine' => 'required|string',
            'prescriptions.*.dosage' => 'nullable|string',
            'prescriptions.*.duration' => 'nullable|string',
            'notes' => 'nullable|string',
        ];
    }

    private function with(): array
    {
        return ['patient:id,patient_id,first_name,last_name', 'doctor:id,name'];
    }

    // GET /api/records?patient_id=&limit=
    public function index(Request $request)
    {
        $records = MedicalRecord::with($this->with())
            ->when($request->patient_id, fn ($q, $v) => $q->where('patient_id', $v))
            ->orderByDesc('visit_date')
            ->paginate($request->integer('limit', 10));

        return response()->json(['success' => true] + $records->toArray());
    }

    public function show(MedicalRecord $record)
    {
        ActivityLogger::log('READ', 'Records', 'Viewed medical record', $record->id);

        return response()->json(['success' => true, 'data' => $record->load($this->with())]);
    }

    public function store(Request $request)
    {
        $data = $request->validate($this->rules(true));
        $data['doctor_id'] = $data['doctor_id'] ?? $request->user()->id;
        $data['visit_date'] = $data['visit_date'] ?? now()->toDateString();

        $record = MedicalRecord::create($data);

        ActivityLogger::log('CREATE', 'Records', 'Created medical record', $record->id, null, $record->toArray());

        return response()->json(['success' => true, 'data' => $record], 201);
    }

    public function update(Request $request, MedicalRecord $record)
    {
        $data = $request->validate($this->rules(false));

        $before = $record->toArray();
        $record->update($data);

        ActivityLogger::log('UPDATE', 'Records', 'Updated medical record',
            $record->id, $before, $record->fresh()->toArray());

        return response()->json(['success' => true, 'data' => $record]);
    }

    public function destroy(MedicalRecord $record)
    {
        $before = $record->toArray();
        $record->delete();

        ActivityLogger::log('DELETE', 'Records', 'Deleted medical record', $record->id, $before);

        return response()->json(['success' => true, 'message' => 'Record deleted']);
    }
}
```

### DashboardController.php

```php
<?php

namespace App\Http\Controllers;

use App\Models\ActivityLog;
use App\Models\Appointment;
use App\Models\Patient;
use App\Models\User;
use Illuminate\Http\Request;

class DashboardController extends Controller
{
    // GET /api/dashboard (read-only summary for the landing screen)
    public function index(Request $request)
    {
        $today = today();

        $data = [
            'total_patients' => Patient::count(),
            'todays_appointments' => Appointment::whereDate('date', $today)->count(),
            'pending_appointments' => Appointment::where('status', 'scheduled')
                ->whereDate('date', '>=', $today)->count(),
            'active_staff' => User::where('status', 'active')->count(),
            'upcoming_appointments' => Appointment::with([
                    'patient:id,patient_id,first_name,last_name',
                    'doctor:id,name',
                ])
                ->where('status', 'scheduled')
                ->whereDate('date', '>=', $today)
                ->orderBy('date')->orderBy('time')
                ->limit(5)->get(),
        ];

        // Recent activity is sensitive: admins only
        if ($request->user()->role === 'admin') {
            $data['recent_activity'] = ActivityLog::latest('created_at')->limit(5)->get();
        }

        return response()->json(['success' => true, 'data' => $data]);
    }
}
```

### ReportController.php

```php
<?php

namespace App\Http\Controllers;

use App\Models\Appointment;
use App\Models\Patient;
use App\Services\ActivityLogger;
use Illuminate\Http\Request;
use Illuminate\Support\Carbon;
use Illuminate\Support\Facades\DB;

class ReportController extends Controller
{
    private function range(Request $request): array
    {
        $request->validate(['from' => 'nullable|date', 'to' => 'nullable|date']);

        $from = $request->from ? Carbon::parse($request->from)->startOfDay() : now()->startOfYear();
        $to = $request->to ? Carbon::parse($request->to)->endOfDay() : now()->endOfDay();

        return [$from, $to];
    }

    // GET /api/reports/patients?from=&to=   (MySQL)
    public function patients(Request $request)
    {
        [$from, $to] = $this->range($request);

        $base = fn () => Patient::whereBetween('created_at', [$from, $to]);

        $data = [
            'from' => $from,
            'to' => $to,
            'total' => $base()->count(),
            'by_gender' => $base()->select('gender', DB::raw('count(*) as count'))
                ->groupBy('gender')->get(),
            'new_patients_per_month' => $base()
                ->select(DB::raw('YEAR(created_at) as year'), DB::raw('MONTH(created_at) as month'), DB::raw('count(*) as count'))
                ->groupBy('year', 'month')->orderBy('year')->orderBy('month')->get(),
        ];

        ActivityLogger::log('READ', 'Reports', 'Generated patient report');

        return response()->json(['success' => true, 'data' => $data]);
    }

    // GET /api/reports/appointments?from=&to=
    public function appointments(Request $request)
    {
        [$from, $to] = $this->range($request);

        $base = fn () => Appointment::whereBetween('date', [$from->toDateString(), $to->toDateString()]);

        $data = [
            'from' => $from,
            'to' => $to,
            'total' => $base()->count(),
            'by_status' => $base()->select('status', DB::raw('count(*) as count'))
                ->groupBy('status')->get(),
            'by_doctor' => $base()
                ->join('users', 'users.id', '=', 'appointments.doctor_id')
                ->select('users.name as doctor', DB::raw('count(*) as count'))
                ->groupBy('users.id', 'users.name')->orderByDesc('count')->get(),
        ];

        ActivityLogger::log('READ', 'Reports', 'Generated appointment report');

        return response()->json(['success' => true, 'data' => $data]);
    }
}
```

### ActivityLogController.php

```php
<?php

namespace App\Http\Controllers;

use App\Models\ActivityLog;
use Illuminate\Http\Request;
use Illuminate\Support\Carbon;

class ActivityLogController extends Controller
{
    // GET /api/logs?user_id=&action=&module=&from=&to=&search=&limit=
    public function index(Request $request)
    {
        $logs = ActivityLog::query()
            ->when($request->user_id, fn ($q, $v) => $q->where('user_id', $v))
            ->when($request->action, fn ($q, $v) => $q->where('action', $v))
            ->when($request->module, fn ($q, $v) => $q->where('module', $v))
            ->when($request->search, fn ($q, $s) => $q->where(
                fn ($w) => $w->where('description', 'like', "%$s%")->orWhere('user_name', 'like', "%$s%")
            ))
            ->when($request->from, fn ($q, $v) => $q->where('created_at', '>=', Carbon::parse($v)->startOfDay()))
            ->when($request->to, fn ($q, $v) => $q->where('created_at', '<=', Carbon::parse($v)->endOfDay()))
            ->latest('created_at')
            ->paginate($request->integer('limit', 20));

        return response()->json(['success' => true] + $logs->toArray());
    }

    public function show(ActivityLog $log)
    {
        return response()->json(['success' => true, 'data' => $log]);
    }
}
```

---

## 8. routes/api.php

```php
<?php

use App\Http\Controllers\ActivityLogController;
use App\Http\Controllers\AppointmentController;
use App\Http\Controllers\AuthController;
use App\Http\Controllers\DashboardController;
use App\Http\Controllers\MedicalRecordController;
use App\Http\Controllers\PatientController;
use App\Http\Controllers\ReportController;
use App\Http\Controllers\UserController;
use Illuminate\Support\Facades\Route;

Route::get('/health', fn () => response()->json(['success' => true, 'status' => 'ok']));

// Public: login (20 attempts per 15 minutes)
Route::post('/auth/login', [AuthController::class, 'login'])->middleware('throttle:20,15');

Route::middleware('auth:sanctum')->group(function () {

    // Auth
    Route::get('/auth/me', [AuthController::class, 'me']);
    Route::post('/auth/logout', [AuthController::class, 'logout']);
    Route::put('/auth/change-password', [AuthController::class, 'changePassword']);

    // Dashboard (Read) - all roles
    Route::get('/dashboard', [DashboardController::class, 'index']);

    // Patients - read/write all roles, delete admin only
    Route::get('/patients', [PatientController::class, 'index']);
    Route::post('/patients', [PatientController::class, 'store']);
    Route::get('/patients/{patient}', [PatientController::class, 'show']);
    Route::put('/patients/{patient}', [PatientController::class, 'update']);
    Route::delete('/patients/{patient}', [PatientController::class, 'destroy'])->middleware('role:admin');

    // Appointments - read all roles
    Route::get('/appointments', [AppointmentController::class, 'index']);
    Route::get('/appointments/{appointment}', [AppointmentController::class, 'show']);
    Route::middleware('role:admin,doctor,staff')->group(function () {
        Route::post('/appointments', [AppointmentController::class, 'store']);
        Route::put('/appointments/{appointment}', [AppointmentController::class, 'update']);
    });
    Route::delete('/appointments/{appointment}', [AppointmentController::class, 'destroy'])
        ->middleware('role:admin,staff');

    // Medical records - clinical staff only
    Route::middleware('role:admin,doctor,nurse')->group(function () {
        Route::get('/records', [MedicalRecordController::class, 'index']);
        Route::post('/records', [MedicalRecordController::class, 'store']);
        Route::get('/records/{record}', [MedicalRecordController::class, 'show']);
        Route::put('/records/{record}', [MedicalRecordController::class, 'update']);
    });
    Route::delete('/records/{record}', [MedicalRecordController::class, 'destroy'])->middleware('role:admin');

    // Reports (Read) - admin, doctor
    Route::middleware('role:admin,doctor')->prefix('reports')->group(function () {
        Route::get('/patients', [ReportController::class, 'patients']);
        Route::get('/appointments', [ReportController::class, 'appointments']);
    });

    // User management - admin only
    Route::middleware('role:admin')->group(function () {
        Route::apiResource('users', UserController::class);

        // Activity logs: read-only, no create/update/delete endpoints
        Route::get('/logs', [ActivityLogController::class, 'index']);
        Route::get('/logs/{log}', [ActivityLogController::class, 'show']);
    });
});
```

---

## 9. database/seeders/DatabaseSeeder.php

```php
<?php

namespace Database\Seeders;

use App\Models\User;
use Illuminate\Database\Seeder;

class DatabaseSeeder extends Seeder
{
    public function run(): void
    {
        $users = [
            ['name' => 'System Admin', 'email' => 'admin@carelink.com', 'password' => 'Admin1234', 'role' => 'admin'],
            ['name' => 'Dr. Maria Santos', 'email' => 'doctor@carelink.com', 'password' => 'Doctor1234', 'role' => 'doctor'],
            ['name' => 'Nurse Joy', 'email' => 'nurse@carelink.com', 'password' => 'Nurse1234', 'role' => 'nurse'],
            ['name' => 'Front Desk', 'email' => 'staff@carelink.com', 'password' => 'Staff1234', 'role' => 'staff'],
        ];

        foreach ($users as $u) {
            User::firstOrCreate(['email' => $u['email']], $u);
        }
    }
}
```

---

## 10. Endpoint summary

| Module | Endpoint | Access |
|---|---|---|
| Auth | POST /api/auth/login, GET /api/auth/me, POST /api/auth/logout, PUT /api/auth/change-password | public / any |
| Dashboard (Read) | GET /api/dashboard | all roles |
| Patients (CRUD) | /api/patients | read/write all, delete admin (soft) |
| Appointments (CRUD) | /api/appointments | write admin/doctor/staff, delete admin/staff |
| Medical Records (CRUD) | /api/records | admin/doctor/nurse, delete admin |
| Reports (Read) | GET /api/reports/patients, /api/reports/appointments | admin, doctor |
| User Management (CRUD) | /api/users | admin only |
| Activity Logs | GET /api/logs, GET /api/logs/{id} | admin only, read-only |

Validation errors come back as 422 with a per-field `errors` object. Every create, update, delete, login, failed login, logout, record view and report run is written to `activity_logs` with user, role, action, module, before/after snapshots, IP and user agent.
