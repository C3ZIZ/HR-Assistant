# HR Assistant Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build an applicant-tracking MVP where HR publishes job postings, candidates apply with a PDF CV, OpenAI scores each CV against the posting, and HR shortlists or rejects with an AI-drafted rejection message.

**Architecture:** Laravel 13 + Inertia v3 + React 19 monolith. Controllers validate through Form Requests and delegate to action classes; AI work sits behind two interfaces (`CvScreener`, `RejectionWriter`) implemented against OpenAI's Responses API with strict JSON schema output. Screening runs on the database queue so applying is never blocked by AI latency. Company isolation is enforced in policies, not controllers.

**Tech Stack:** Laravel 13, PHP 8.4, Inertia v3, React 19, TypeScript, Wayfinder, Fortify, spatie/laravel-permission 8, openai-php/laravel, smalot/pdfparser, shadcn/ui, Tailwind v4, Pest 5, SQLite.

**Spec:** `docs/superpowers/specs/2026-08-12-hr-assistant-design.md`

## Global Constraints

- **Never name the model `Job` or the table `jobs`.** Laravel owns `jobs` for the database queue, which this app uses. The domain model is `JobPosting` / `job_postings`.
- PHP 8.4. Curly braces on all control structures. Constructor property promotion. Explicit return types and parameter type hints on every method. TitleCase enum cases. PHPDoc blocks over inline comments, with array shape types.
- Create files with `php artisan make:*` and always pass `--no-interaction`.
- Run `vendor/bin/pint --dirty --format agent` after touching PHP. Never `pint --test`.
- Every change ships with a test. Run `php artisan test --compact --filter=<name>`.
- Roles are global; company isolation is a `company_id` comparison inside policies. Spatie's teams feature stays **off**.
- CVs live on the private `local` disk. Never the `public` disk.
- `OPENAI_MODEL` has **no default**. The screener throws if it is unset.
- All frontend URLs come from Wayfinder imports (`@/actions/...`, `@/routes/...`). No hardcoded URL strings.

## File Structure

**Backend — created**

```
app/Enums/{ApplicationStatus,ScreeningStatus,JobPostingStatus,EmploymentType,RoleName}.php
app/Models/{Company,CompanyInvitation,JobPosting,Application,ApplicationScreening}.php
app/Contracts/{CvScreener,RejectionWriter}.php
app/Data/ScreeningResult.php
app/Exceptions/{CvExtractionException,AiResponseException}.php
app/Services/Cv/CvTextExtractor.php
app/Services/Ai/{OpenAiCvScreener,OpenAiRejectionWriter}.php
app/Actions/Companies/{RegisterCompany,InviteTeamMember,AcceptInvitation}.php
app/Actions/Applications/{SubmitApplication,ShortlistApplication,RejectApplication}.php
app/Actions/Fortify/CreateNewUser.php
app/Jobs/ScreenApplicationJob.php
app/Policies/{JobPostingPolicy,ApplicationPolicy,CompanyPolicy}.php
app/Http/Controllers/{DashboardController,JobBoardController}.php
app/Http/Controllers/Auth/{RegisteredCompanyController,InvitationController}.php
app/Http/Controllers/Hr/{JobPostingController,ApplicationController,ApplicationDecisionController,RejectionMessageController,TeamController,CvDownloadController,RescreenController}.php
app/Http/Controllers/Candidate/ApplicationController.php
app/Http/Requests/**  (one per write endpoint)
config/hr.php
routes/hr.php
database/migrations/**  (6 new)
database/factories/{CompanyFactory,JobPostingFactory,ApplicationFactory,ApplicationScreeningFactory}.php
database/seeders/{RolesAndPermissionsSeeder,DemoSeeder}.php
tests/Support/{FakeCvScreener,FakeRejectionWriter}.php
tests/Feature/**  (one file per slice)
```

**Backend — modified**

- `app/Models/User.php` — `HasRoles` trait, `company_id`, relations
- `app/Providers/AppServiceProvider.php` — bind the two AI contracts
- `app/Providers/FortifyServiceProvider.php` — register view + `createUsersUsing`
- `app/Http/Middleware/HandleInertiaRequests.php` — share roles and company
- `bootstrap/app.php` — spatie middleware aliases
- `config/fortify.php` — enable registration
- `routes/web.php` — job board, dashboard, require `hr.php`
- `tests/Pest.php` — seed roles, reset permission cache
- `tests/Feature/DashboardTest.php` — dashboard is now a redirect

**Frontend — created**

```
resources/js/layouts/public-layout.tsx
resources/js/components/match-score-badge.tsx
resources/js/components/screening-panel.tsx
resources/js/components/reject-dialog.tsx
resources/js/components/job-posting-form.tsx
resources/js/pages/auth/{register,register-company,accept-invitation}.tsx
resources/js/pages/jobs/{index,show}.tsx
resources/js/pages/candidate/applications.tsx
resources/js/pages/hr/jobs/{index,create,edit,applications}.tsx
resources/js/pages/hr/applications/show.tsx
resources/js/pages/hr/team.tsx
resources/js/types/hr.ts
```

**Frontend — modified**

- `resources/js/app.tsx` — layout resolver for `jobs/`
- `resources/js/components/app-sidebar.tsx` — role-aware nav
- `resources/js/types/auth.ts` — roles + company on `Auth`

---

## Task 0: Repository, dependencies, and configuration

**Files:**
- Create: `config/hr.php`
- Modify: `bootstrap/app.php`, `.env`, `.env.example`, `app/Models/User.php`
- Test: `tests/Feature/ConfigurationTest.php`

**Interfaces:**
- Consumes: nothing
- Produces: `config('hr.ai.model')`, `config('hr.ai.min_cv_characters')`, `config('hr.ai.max_cv_characters')`, `config('hr.cv.disk')`, `config('hr.cv.max_kilobytes')`, `config('hr.invitations.expires_after_days')`; the `role:` and `permission:` middleware aliases; `User` has the `HasRoles` trait.

- [ ] **Step 1: Initialise git**

This project is not yet a git repository, so nothing below can be committed until it is.

```bash
git init
git add -A
git commit -m "chore: initial commit of Laravel React starter kit"
```

- [ ] **Step 2: Install the three packages**

```bash
composer require spatie/laravel-permission:^8.3 openai-php/laravel:^0.20 smalot/pdfparser:^2.12 --no-interaction
```

- [ ] **Step 3: Publish spatie's config and migrations**

```bash
php artisan vendor:publish --provider="Spatie\Permission\PermissionServiceProvider" --no-interaction
php artisan optimize:clear
```

- [ ] **Step 4: Publish the OpenAI config**

```bash
php artisan openai:install --no-interaction
```

- [ ] **Step 5: Add environment variables**

Append to **`.env`** (replace the key with your real one):

```env
OPENAI_API_KEY=sk-your-real-key-here
OPENAI_MODEL=gpt-5.6
```

Append to **`.env.example`** (no value — the model is a deliberate per-environment choice):

```env
OPENAI_API_KEY=
# Model used for CV screening and rejection letters, e.g. gpt-5.6
OPENAI_MODEL=
```

`php artisan openai:install` already appended `OPENAI_API_KEY` and `OPENAI_ORGANIZATION` to `.env`. Delete the duplicate `OPENAI_API_KEY` line if there is one, and leave `OPENAI_ORGANIZATION` blank unless your account requires it.

- [ ] **Step 6: Create the application config file**

Create `config/hr.php`:

```php
<?php

return [

    /*
    |--------------------------------------------------------------------------
    | AI Screening
    |--------------------------------------------------------------------------
    |
    | The model has no default on purpose: it is an account-specific choice and
    | a wrong guess fails at request time with a confusing provider error. The
    | screener throws immediately when it is missing instead.
    |
    */

    'ai' => [
        'model' => env('OPENAI_MODEL'),
        'min_cv_characters' => 200,
        'max_cv_characters' => 20000,
    ],

    /*
    |--------------------------------------------------------------------------
    | CV Storage
    |--------------------------------------------------------------------------
    */

    'cv' => [
        'disk' => env('HR_CV_DISK', 'local'),
        'max_kilobytes' => 5120,
    ],

    /*
    |--------------------------------------------------------------------------
    | Team Invitations
    |--------------------------------------------------------------------------
    */

    'invitations' => [
        'expires_after_days' => 7,
    ],

];
```

- [ ] **Step 7: Register spatie's middleware aliases**

In `bootstrap/app.php`, replace the whole `->withMiddleware(...)` closure with:

```php
    ->withMiddleware(function (Middleware $middleware): void {
        $middleware->encryptCookies(except: ['appearance', 'sidebar_state']);

        $middleware->alias([
            'role' => \Spatie\Permission\Middleware\RoleMiddleware::class,
            'permission' => \Spatie\Permission\Middleware\PermissionMiddleware::class,
        ]);

        $middleware->web(append: [
            HandleAppearance::class,
            HandleInertiaRequests::class,
            AddLinkHeadersForPreloadedAssets::class,
        ]);
    })
```

- [ ] **Step 8: Add the HasRoles trait to User**

In `app/Models/User.php`, add the import and the trait. Leave everything else alone for now — `company_id` and the relations come in Task 1.

```php
use Spatie\Permission\Traits\HasRoles;
```

```php
    /** @use HasFactory<UserFactory> */
    use HasFactory, HasRoles, Notifiable;
```

- [ ] **Step 9: Write the failing test**

Create `tests/Feature/ConfigurationTest.php`:

```php
<?php

use Spatie\Permission\Middleware\RoleMiddleware;

test('hr configuration is registered', function () {
    expect(config('hr.cv.disk'))->toBe('local')
        ->and(config('hr.cv.max_kilobytes'))->toBe(5120)
        ->and(config('hr.ai.min_cv_characters'))->toBe(200)
        ->and(config('hr.ai.max_cv_characters'))->toBe(20000)
        ->and(config('hr.invitations.expires_after_days'))->toBe(7);
});

test('the role middleware alias is registered', function () {
    $aliases = app('router')->getMiddleware();

    expect($aliases)->toHaveKey('role')
        ->and($aliases['role'])->toBe(RoleMiddleware::class);
});

test('the user model can hold roles', function () {
    expect(method_exists(App\Models\User::class, 'assignRole'))->toBeTrue();
});
```

- [ ] **Step 10: Run the test**

```bash
php artisan test --compact --filter=ConfigurationTest
```

Expected: PASS (3 tests). A failure on `role` means Step 7 was not applied.

- [ ] **Step 11: Run migrations and format**

```bash
php artisan migrate
vendor/bin/pint --dirty --format agent
```

- [ ] **Step 12: Commit**

```bash
git add -A
git commit -m "chore: install permission, openai and pdfparser packages with hr config"
```

---

## Task 1: Enums, schema, models, factories, roles

**Files:**
- Create: `app/Enums/*.php` (5), 6 migrations, `app/Models/*.php` (5), 4 factories, `database/seeders/RolesAndPermissionsSeeder.php`
- Modify: `app/Models/User.php`, `database/seeders/DatabaseSeeder.php`, `tests/Pest.php`
- Test: `tests/Feature/ModelsTest.php`

**Interfaces:**
- Consumes: Task 0's `HasRoles` trait and `config('hr.*')`
- Produces:
  - Enums `ApplicationStatus{Submitted,Shortlisted,Rejected}`, `ScreeningStatus{Pending,Completed,Failed}`, `JobPostingStatus{Draft,Published,Closed}`, `EmploymentType{FullTime,PartTime,Contract,Internship}`, `RoleName{CompanyAdmin,Hr,Candidate}`, all backed by strings, all with `label(): string`
  - `Company` hasMany `users`, `jobPostings`, `invitations`
  - `JobPosting` belongsTo `company`, `creator`; hasMany `applications`; `scopePublished()`
  - `Application` belongsTo `jobPosting`, `candidate`, `decider`; hasOne `latestScreening`
  - `ApplicationScreening` belongsTo `application`
  - `User` belongsTo `company`; hasMany `applications`
  - Factories `CompanyFactory`, `JobPostingFactory` (states `published()`, `draft()`), `ApplicationFactory` (states `shortlisted()`, `rejected()`), `ApplicationScreeningFactory` (states `completed()`, `failed()`)
  - Seeder `RolesAndPermissionsSeeder`

- [ ] **Step 1: Generate the enums**

```bash
php artisan make:enum Enums/ApplicationStatus --string --no-interaction
php artisan make:enum Enums/ScreeningStatus --string --no-interaction
php artisan make:enum Enums/JobPostingStatus --string --no-interaction
php artisan make:enum Enums/EmploymentType --string --no-interaction
php artisan make:enum Enums/RoleName --string --no-interaction
php artisan make:enum Enums/PermissionName --string --no-interaction
```

- [ ] **Step 2: Write the enums**

`app/Enums/ApplicationStatus.php`:

```php
<?php

namespace App\Enums;

enum ApplicationStatus: string
{
    case Submitted = 'submitted';
    case Shortlisted = 'shortlisted';
    case Rejected = 'rejected';

    public function label(): string
    {
        return match ($this) {
            self::Submitted => 'Submitted',
            self::Shortlisted => 'Shortlisted',
            self::Rejected => 'Rejected',
        };
    }

    public function isDecided(): bool
    {
        return $this !== self::Submitted;
    }
}
```

`app/Enums/ScreeningStatus.php`:

```php
<?php

namespace App\Enums;

enum ScreeningStatus: string
{
    case Pending = 'pending';
    case Completed = 'completed';
    case Failed = 'failed';

    public function label(): string
    {
        return match ($this) {
            self::Pending => 'Screening',
            self::Completed => 'Screened',
            self::Failed => 'Screening failed',
        };
    }
}
```

`app/Enums/JobPostingStatus.php`:

```php
<?php

namespace App\Enums;

enum JobPostingStatus: string
{
    case Draft = 'draft';
    case Published = 'published';
    case Closed = 'closed';

    public function label(): string
    {
        return match ($this) {
            self::Draft => 'Draft',
            self::Published => 'Published',
            self::Closed => 'Closed',
        };
    }
}
```

`app/Enums/EmploymentType.php`:

```php
<?php

namespace App\Enums;

enum EmploymentType: string
{
    case FullTime = 'full_time';
    case PartTime = 'part_time';
    case Contract = 'contract';
    case Internship = 'internship';

    public function label(): string
    {
        return match ($this) {
            self::FullTime => 'Full time',
            self::PartTime => 'Part time',
            self::Contract => 'Contract',
            self::Internship => 'Internship',
        };
    }
}
```

`app/Enums/RoleName.php`:

```php
<?php

namespace App\Enums;

enum RoleName: string
{
    case CompanyAdmin = 'company_admin';
    case Hr = 'hr';
    case Candidate = 'candidate';

    public function label(): string
    {
        return match ($this) {
            self::CompanyAdmin => 'Company admin',
            self::Hr => 'HR',
            self::Candidate => 'Candidate',
        };
    }

    /**
     * Roles allowed to reach the HR area.
     *
     * @return array<int, string>
     */
    public static function hrRoles(): array
    {
        return [self::CompanyAdmin->value, self::Hr->value];
    }
}
```

`app/Enums/PermissionName.php` — named `PermissionName`, not `Permission`, to avoid colliding with `Spatie\Permission\Models\Permission` in imports:

```php
<?php

namespace App\Enums;

enum PermissionName: string
{
    case ManageJobs = 'manage jobs';
    case ReviewApplications = 'review applications';
    case ManageTeam = 'manage team';

    /**
     * @return array<int, string>
     */
    public static function values(): array
    {
        return array_column(self::cases(), 'value');
    }
}
```

- [ ] **Step 3: Generate the models with their migrations and factories**

```bash
php artisan make:model Company -mf --no-interaction
php artisan make:model CompanyInvitation -mf --no-interaction
php artisan make:model JobPosting -mf --no-interaction
php artisan make:model Application -mf --no-interaction
php artisan make:model ApplicationScreening -mf --no-interaction
php artisan make:migration add_company_id_to_users_table --no-interaction
```

The `add_company_id_to_users_table` migration must have a **later** timestamp than the companies migration. Because you generated them in this order it already does — confirm with `ls database/migrations`.

- [ ] **Step 4: Write the migrations**

`..._create_companies_table.php`:

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('companies', function (Blueprint $table) {
            $table->id();
            $table->string('name');
            $table->string('slug')->unique();
            $table->timestamps();
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('companies');
    }
};
```

`..._create_company_invitations_table.php`:

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('company_invitations', function (Blueprint $table) {
            $table->id();
            $table->foreignId('company_id')->constrained()->cascadeOnDelete();
            $table->foreignId('invited_by')->constrained('users')->cascadeOnDelete();
            $table->string('email');
            $table->string('token', 64)->unique();
            $table->timestamp('expires_at');
            $table->timestamp('accepted_at')->nullable();
            $table->timestamps();

            $table->index(['company_id', 'email']);
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('company_invitations');
    }
};
```

`..._create_job_postings_table.php`:

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('job_postings', function (Blueprint $table) {
            $table->id();
            $table->foreignId('company_id')->constrained()->cascadeOnDelete();
            $table->foreignId('created_by')->constrained('users')->cascadeOnDelete();
            $table->string('title');
            $table->text('description');
            $table->text('requirements');
            $table->string('employment_type');
            $table->string('location');
            $table->unsignedTinyInteger('min_years_experience')->default(0);
            $table->string('status');
            $table->timestamp('published_at')->nullable();
            $table->timestamps();

            $table->index(['company_id', 'status']);
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('job_postings');
    }
};
```

`..._create_applications_table.php`:

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('applications', function (Blueprint $table) {
            $table->id();
            $table->foreignId('job_posting_id')->constrained()->cascadeOnDelete();
            $table->foreignId('user_id')->constrained()->cascadeOnDelete();
            $table->string('cv_path');
            $table->string('cv_original_name');
            $table->longText('cv_text')->nullable();
            $table->string('status');
            $table->text('rejection_reason')->nullable();
            $table->timestamp('decided_at')->nullable();
            $table->foreignId('decided_by')->nullable()->constrained('users')->nullOnDelete();
            $table->timestamps();

            $table->unique(['job_posting_id', 'user_id']);
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('applications');
    }
};
```

`..._create_application_screenings_table.php`:

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('application_screenings', function (Blueprint $table) {
            $table->id();
            $table->foreignId('application_id')->constrained()->cascadeOnDelete();
            $table->string('status');
            $table->unsignedTinyInteger('match_score')->nullable();
            $table->text('summary')->nullable();
            $table->json('strengths')->nullable();
            $table->json('gaps')->nullable();
            $table->string('recommendation')->nullable();
            $table->string('model')->nullable();
            $table->unsignedInteger('input_tokens')->nullable();
            $table->unsignedInteger('output_tokens')->nullable();
            $table->text('error')->nullable();
            $table->timestamp('completed_at')->nullable();
            $table->timestamps();
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('application_screenings');
    }
};
```

`..._add_company_id_to_users_table.php`:

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::table('users', function (Blueprint $table) {
            $table->foreignId('company_id')->nullable()->after('id')->constrained()->nullOnDelete();
        });
    }

    public function down(): void
    {
        Schema::table('users', function (Blueprint $table) {
            $table->dropConstrainedForeignId('company_id');
        });
    }
};
```

- [ ] **Step 5: Write the models**

`app/Models/Company.php`:

```php
<?php

namespace App\Models;

use Database\Factories\CompanyFactory;
use Illuminate\Database\Eloquent\Attributes\Fillable;
use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\HasMany;
use Illuminate\Support\Carbon;

/**
 * @property int $id
 * @property string $name
 * @property string $slug
 * @property Carbon|null $created_at
 * @property Carbon|null $updated_at
 */
#[Fillable(['name', 'slug'])]
class Company extends Model
{
    /** @use HasFactory<CompanyFactory> */
    use HasFactory;

    /** @return HasMany<User, $this> */
    public function users(): HasMany
    {
        return $this->hasMany(User::class);
    }

    /** @return HasMany<JobPosting, $this> */
    public function jobPostings(): HasMany
    {
        return $this->hasMany(JobPosting::class);
    }

    /** @return HasMany<CompanyInvitation, $this> */
    public function invitations(): HasMany
    {
        return $this->hasMany(CompanyInvitation::class);
    }
}
```

`app/Models/CompanyInvitation.php`:

```php
<?php

namespace App\Models;

use Database\Factories\CompanyInvitationFactory;
use Illuminate\Database\Eloquent\Attributes\Fillable;
use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsTo;
use Illuminate\Support\Carbon;

/**
 * @property int $id
 * @property int $company_id
 * @property int $invited_by
 * @property string $email
 * @property string $token
 * @property Carbon $expires_at
 * @property Carbon|null $accepted_at
 */
#[Fillable(['company_id', 'invited_by', 'email', 'token', 'expires_at'])]
class CompanyInvitation extends Model
{
    /** @use HasFactory<CompanyInvitationFactory> */
    use HasFactory;

    /** @return array<string, string> */
    protected function casts(): array
    {
        return [
            'expires_at' => 'datetime',
            'accepted_at' => 'datetime',
        ];
    }

    /** @return BelongsTo<Company, $this> */
    public function company(): BelongsTo
    {
        return $this->belongsTo(Company::class);
    }

    public function isPending(): bool
    {
        return $this->accepted_at === null && $this->expires_at->isFuture();
    }
}
```

`app/Models/JobPosting.php`:

```php
<?php

namespace App\Models;

use App\Enums\EmploymentType;
use App\Enums\JobPostingStatus;
use Database\Factories\JobPostingFactory;
use Illuminate\Database\Eloquent\Attributes\Fillable;
use Illuminate\Database\Eloquent\Builder;
use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsTo;
use Illuminate\Database\Eloquent\Relations\HasMany;
use Illuminate\Support\Carbon;

/**
 * @property int $id
 * @property int $company_id
 * @property int $created_by
 * @property string $title
 * @property string $description
 * @property string $requirements
 * @property EmploymentType $employment_type
 * @property string $location
 * @property int $min_years_experience
 * @property JobPostingStatus $status
 * @property Carbon|null $published_at
 */
#[Fillable([
    'title',
    'description',
    'requirements',
    'employment_type',
    'location',
    'min_years_experience',
    'status',
])]
class JobPosting extends Model
{
    /** @use HasFactory<JobPostingFactory> */
    use HasFactory;

    /** @return array<string, string> */
    protected function casts(): array
    {
        return [
            'employment_type' => EmploymentType::class,
            'status' => JobPostingStatus::class,
            'published_at' => 'datetime',
        ];
    }

    /** @return BelongsTo<Company, $this> */
    public function company(): BelongsTo
    {
        return $this->belongsTo(Company::class);
    }

    /** @return BelongsTo<User, $this> */
    public function creator(): BelongsTo
    {
        return $this->belongsTo(User::class, 'created_by');
    }

    /** @return HasMany<Application, $this> */
    public function applications(): HasMany
    {
        return $this->hasMany(Application::class);
    }

    /** @param Builder<JobPosting> $query */
    public function scopePublished(Builder $query): void
    {
        $query->where('status', JobPostingStatus::Published);
    }

    public function isOpen(): bool
    {
        return $this->status === JobPostingStatus::Published;
    }
}
```

`app/Models/Application.php`:

```php
<?php

namespace App\Models;

use App\Enums\ApplicationStatus;
use Database\Factories\ApplicationFactory;
use Illuminate\Database\Eloquent\Attributes\Fillable;
use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsTo;
use Illuminate\Database\Eloquent\Relations\HasMany;
use Illuminate\Database\Eloquent\Relations\HasOne;
use Illuminate\Support\Carbon;

/**
 * @property int $id
 * @property int $job_posting_id
 * @property int $user_id
 * @property string $cv_path
 * @property string $cv_original_name
 * @property string|null $cv_text
 * @property ApplicationStatus $status
 * @property string|null $rejection_reason
 * @property Carbon|null $decided_at
 * @property int|null $decided_by
 */
#[Fillable(['job_posting_id', 'user_id', 'cv_path', 'cv_original_name', 'status'])]
class Application extends Model
{
    /** @use HasFactory<ApplicationFactory> */
    use HasFactory;

    /** @return array<string, string> */
    protected function casts(): array
    {
        return [
            'status' => ApplicationStatus::class,
            'decided_at' => 'datetime',
        ];
    }

    /** @return BelongsTo<JobPosting, $this> */
    public function jobPosting(): BelongsTo
    {
        return $this->belongsTo(JobPosting::class);
    }

    /** @return BelongsTo<User, $this> */
    public function candidate(): BelongsTo
    {
        return $this->belongsTo(User::class, 'user_id');
    }

    /** @return BelongsTo<User, $this> */
    public function decider(): BelongsTo
    {
        return $this->belongsTo(User::class, 'decided_by');
    }

    /** @return HasMany<ApplicationScreening, $this> */
    public function screenings(): HasMany
    {
        return $this->hasMany(ApplicationScreening::class);
    }

    /** @return HasOne<ApplicationScreening, $this> */
    public function latestScreening(): HasOne
    {
        return $this->hasOne(ApplicationScreening::class)->latestOfMany();
    }
}
```

`app/Models/ApplicationScreening.php`:

```php
<?php

namespace App\Models;

use App\Enums\ScreeningStatus;
use Database\Factories\ApplicationScreeningFactory;
use Illuminate\Database\Eloquent\Attributes\Fillable;
use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsTo;
use Illuminate\Support\Carbon;

/**
 * @property int $id
 * @property int $application_id
 * @property ScreeningStatus $status
 * @property int|null $match_score
 * @property string|null $summary
 * @property array<int, string>|null $strengths
 * @property array<int, string>|null $gaps
 * @property string|null $recommendation
 * @property string|null $model
 * @property int|null $input_tokens
 * @property int|null $output_tokens
 * @property string|null $error
 * @property Carbon|null $completed_at
 */
#[Fillable([
    'application_id',
    'status',
    'match_score',
    'summary',
    'strengths',
    'gaps',
    'recommendation',
    'model',
    'input_tokens',
    'output_tokens',
    'error',
    'completed_at',
])]
class ApplicationScreening extends Model
{
    /** @use HasFactory<ApplicationScreeningFactory> */
    use HasFactory;

    /** @return array<string, string> */
    protected function casts(): array
    {
        return [
            'status' => ScreeningStatus::class,
            'strengths' => 'array',
            'gaps' => 'array',
            'completed_at' => 'datetime',
        ];
    }

    /** @return BelongsTo<Application, $this> */
    public function application(): BelongsTo
    {
        return $this->belongsTo(Application::class);
    }
}
```

- [ ] **Step 6: Extend the User model**

In `app/Models/User.php`, add these imports:

```php
use Illuminate\Database\Eloquent\Relations\BelongsTo;
use Illuminate\Database\Eloquent\Relations\HasMany;
```

Add `@property int|null $company_id` to the class docblock, and append these two relations inside the class:

```php
    /** @return BelongsTo<Company, $this> */
    public function company(): BelongsTo
    {
        return $this->belongsTo(Company::class);
    }

    /** @return HasMany<Application, $this> */
    public function applications(): HasMany
    {
        return $this->hasMany(Application::class);
    }
```

**Leave `company_id` out of `#[Fillable]`.** It is assigned explicitly by the action classes. Adding it to fillable would let a crafted registration payload place a new user inside somebody else's company.

- [ ] **Step 7: Write the factories**

`database/factories/CompanyFactory.php`:

```php
<?php

namespace Database\Factories;

use App\Models\Company;
use Illuminate\Database\Eloquent\Factories\Factory;
use Illuminate\Support\Str;

/**
 * @extends Factory<Company>
 */
class CompanyFactory extends Factory
{
    /**
     * @return array<string, mixed>
     */
    public function definition(): array
    {
        $name = fake()->unique()->company();

        return [
            'name' => $name,
            'slug' => Str::slug($name).'-'.Str::lower(Str::random(6)),
        ];
    }
}
```

`database/factories/CompanyInvitationFactory.php`:

```php
<?php

namespace Database\Factories;

use App\Models\Company;
use App\Models\CompanyInvitation;
use App\Models\User;
use Illuminate\Database\Eloquent\Factories\Factory;
use Illuminate\Support\Str;

/**
 * @extends Factory<CompanyInvitation>
 */
class CompanyInvitationFactory extends Factory
{
    /**
     * @return array<string, mixed>
     */
    public function definition(): array
    {
        return [
            'company_id' => Company::factory(),
            'invited_by' => User::factory(),
            'email' => fake()->unique()->safeEmail(),
            'token' => Str::random(64),
            'expires_at' => now()->addDays(7),
        ];
    }

    public function expired(): static
    {
        return $this->state(fn (array $attributes) => [
            'expires_at' => now()->subDay(),
        ]);
    }

    public function accepted(): static
    {
        return $this->state(fn (array $attributes) => [
            'accepted_at' => now(),
        ]);
    }
}
```

`database/factories/JobPostingFactory.php`:

```php
<?php

namespace Database\Factories;

use App\Enums\EmploymentType;
use App\Enums\JobPostingStatus;
use App\Models\Company;
use App\Models\JobPosting;
use App\Models\User;
use Illuminate\Database\Eloquent\Factories\Factory;

/**
 * @extends Factory<JobPosting>
 */
class JobPostingFactory extends Factory
{
    /**
     * @return array<string, mixed>
     */
    public function definition(): array
    {
        return [
            'company_id' => Company::factory(),
            'created_by' => User::factory(),
            'title' => fake()->jobTitle(),
            'description' => fake()->paragraphs(3, true),
            'requirements' => fake()->paragraphs(2, true),
            'employment_type' => fake()->randomElement(EmploymentType::cases()),
            'location' => fake()->city(),
            'min_years_experience' => fake()->numberBetween(0, 8),
            'status' => JobPostingStatus::Draft,
        ];
    }

    public function published(): static
    {
        return $this->state(fn (array $attributes) => [
            'status' => JobPostingStatus::Published,
            'published_at' => now(),
        ]);
    }

    public function closed(): static
    {
        return $this->state(fn (array $attributes) => [
            'status' => JobPostingStatus::Closed,
        ]);
    }
}
```

`database/factories/ApplicationFactory.php`:

```php
<?php

namespace Database\Factories;

use App\Enums\ApplicationStatus;
use App\Models\Application;
use App\Models\JobPosting;
use App\Models\User;
use Illuminate\Database\Eloquent\Factories\Factory;

/**
 * @extends Factory<Application>
 */
class ApplicationFactory extends Factory
{
    /**
     * @return array<string, mixed>
     */
    public function definition(): array
    {
        return [
            'job_posting_id' => JobPosting::factory(),
            'user_id' => User::factory(),
            'cv_path' => 'cvs/1/1/'.fake()->uuid().'.pdf',
            'cv_original_name' => 'resume.pdf',
            'cv_text' => fake()->paragraphs(5, true),
            'status' => ApplicationStatus::Submitted,
        ];
    }

    public function shortlisted(): static
    {
        return $this->state(fn (array $attributes) => [
            'status' => ApplicationStatus::Shortlisted,
            'decided_at' => now(),
        ]);
    }

    public function rejected(): static
    {
        return $this->state(fn (array $attributes) => [
            'status' => ApplicationStatus::Rejected,
            'rejection_reason' => 'Thank you for applying. We have decided to move forward with other candidates.',
            'decided_at' => now(),
        ]);
    }
}
```

`database/factories/ApplicationScreeningFactory.php`:

```php
<?php

namespace Database\Factories;

use App\Enums\ScreeningStatus;
use App\Models\Application;
use App\Models\ApplicationScreening;
use Illuminate\Database\Eloquent\Factories\Factory;

/**
 * @extends Factory<ApplicationScreening>
 */
class ApplicationScreeningFactory extends Factory
{
    /**
     * @return array<string, mixed>
     */
    public function definition(): array
    {
        return [
            'application_id' => Application::factory(),
            'status' => ScreeningStatus::Pending,
        ];
    }

    public function completed(): static
    {
        return $this->state(fn (array $attributes) => [
            'status' => ScreeningStatus::Completed,
            'match_score' => fake()->numberBetween(10, 95),
            'summary' => fake()->sentence(20),
            'strengths' => [fake()->sentence(), fake()->sentence()],
            'gaps' => [fake()->sentence()],
            'recommendation' => fake()->randomElement(['advance', 'hold', 'reject']),
            'model' => 'test-model',
            'input_tokens' => 1200,
            'output_tokens' => 240,
            'completed_at' => now(),
        ]);
    }

    public function failed(): static
    {
        return $this->state(fn (array $attributes) => [
            'status' => ScreeningStatus::Failed,
            'error' => 'The screening provider returned an error.',
        ]);
    }
}
```

- [ ] **Step 8: Write the roles seeder**

```bash
php artisan make:seeder RolesAndPermissionsSeeder --no-interaction
```

`database/seeders/RolesAndPermissionsSeeder.php`:

```php
<?php

namespace Database\Seeders;

use App\Enums\PermissionName;
use App\Enums\RoleName;
use Illuminate\Database\Seeder;
use Spatie\Permission\Models\Permission;
use Spatie\Permission\Models\Role;
use Spatie\Permission\PermissionRegistrar;

class RolesAndPermissionsSeeder extends Seeder
{
    /**
     * Seed roles and permissions. Safe to run repeatedly.
     */
    public function run(): void
    {
        app(PermissionRegistrar::class)->forgetCachedPermissions();

        foreach (PermissionName::values() as $permission) {
            Permission::findOrCreate($permission, 'web');
        }

        Role::findOrCreate(RoleName::CompanyAdmin->value, 'web')->syncPermissions([
            PermissionName::ManageJobs->value,
            PermissionName::ReviewApplications->value,
            PermissionName::ManageTeam->value,
        ]);

        Role::findOrCreate(RoleName::Hr->value, 'web')->syncPermissions([
            PermissionName::ManageJobs->value,
            PermissionName::ReviewApplications->value,
        ]);

        Role::findOrCreate(RoleName::Candidate->value, 'web')->syncPermissions([]);
    }
}
```

- [ ] **Step 9: Call the seeder from DatabaseSeeder**

Replace the body of `run()` in `database/seeders/DatabaseSeeder.php` with:

```php
    public function run(): void
    {
        $this->call(RolesAndPermissionsSeeder::class);
    }
```

- [ ] **Step 10: Make roles available to every feature test**

Spatie caches permissions, and `RefreshDatabase` wipes the tables underneath that cache. Without this hook you get intermittent "There is no role named company_admin" failures. Replace the `pest()->extend(...)` block in `tests/Pest.php` with:

```php
pest()->extend(TestCase::class)
    ->use(RefreshDatabase::class)
    ->beforeEach(function () {
        app(\Spatie\Permission\PermissionRegistrar::class)->forgetCachedPermissions();
        $this->seed(\Database\Seeders\RolesAndPermissionsSeeder::class);
    })
    ->in('Feature');
```

- [ ] **Step 11: Write the failing test**

Create `tests/Feature/ModelsTest.php`:

```php
<?php

use App\Enums\ApplicationStatus;
use App\Enums\JobPostingStatus;
use App\Enums\RoleName;
use App\Models\Application;
use App\Models\ApplicationScreening;
use App\Models\Company;
use App\Models\JobPosting;
use App\Models\User;

test('a company has users and job postings', function () {
    $company = Company::factory()->create();
    $user = User::factory()->create(['company_id' => $company->id]);
    JobPosting::factory()->for($company)->create(['created_by' => $user->id]);

    expect($company->users)->toHaveCount(1)
        ->and($company->jobPostings)->toHaveCount(1)
        ->and($user->company->id)->toBe($company->id);
});

test('the published scope only returns published postings', function () {
    JobPosting::factory()->create();
    JobPosting::factory()->published()->create();

    expect(JobPosting::published()->count())->toBe(1);
});

test('an application exposes its latest screening', function () {
    $application = Application::factory()->create();
    ApplicationScreening::factory()->for($application)->failed()->create();
    $latest = ApplicationScreening::factory()->for($application)->completed()->create();

    expect($application->fresh()->latestScreening->id)->toBe($latest->id)
        ->and($application->latestScreening->strengths)->toBeArray();
});

test('enums cast on their models', function () {
    $application = Application::factory()->rejected()->create();

    expect($application->status)->toBe(ApplicationStatus::Rejected)
        ->and($application->jobPosting->status)->toBe(JobPostingStatus::Draft);
});

test('the seeder creates the three roles with their permissions', function () {
    $admin = User::factory()->create();
    $admin->assignRole(RoleName::CompanyAdmin->value);

    expect($admin->can('manage jobs'))->toBeTrue()
        ->and($admin->can('manage team'))->toBeTrue();

    $hr = User::factory()->create();
    $hr->assignRole(RoleName::Hr->value);

    expect($hr->can('review applications'))->toBeTrue()
        ->and($hr->can('manage team'))->toBeFalse();
});
```

- [ ] **Step 12: Run migrations, then the test**

```bash
php artisan migrate
php artisan test --compact --filter=ModelsTest
```

Expected: PASS (5 tests).

- [ ] **Step 13: Format and commit**

```bash
vendor/bin/pint --dirty --format agent
git add -A
git commit -m "feat: add hr domain schema, models, factories and roles"
```

---

## Task 2: Candidate and company registration

**Files:**
- Create: `app/Actions/Fortify/CreateNewUser.php`, `app/Actions/Companies/RegisterCompany.php`, `app/Http/Controllers/Auth/RegisteredCompanyController.php`, `app/Http/Requests/Auth/RegisterCompanyRequest.php`, `app/Http/Controllers/DashboardController.php`, `resources/js/pages/auth/register.tsx`, `resources/js/pages/auth/register-company.tsx`
- Modify: `config/fortify.php`, `app/Providers/FortifyServiceProvider.php`, `routes/web.php`, `app/Http/Middleware/HandleInertiaRequests.php`, `resources/js/types/auth.ts`, `tests/Feature/DashboardTest.php`
- Test: `tests/Feature/Auth/RegistrationTest.php`, `tests/Feature/Auth/CompanyRegistrationTest.php`

**Interfaces:**
- Consumes: `RoleName`, `Company`, `User` from Task 1
- Produces:
  - `RegisterCompany::handle(string $companyName, string $name, string $email, string $password): User` — returns the persisted admin user with the `company_admin` role and `company_id` set
  - Route names `register`, `register.company`, `register.company.store`, `dashboard`
  - Shared Inertia props `auth.roles` (`string[]`) and `auth.company` (`{id, name} | null`)

- [ ] **Step 1: Write the failing tests**

Create `tests/Feature/Auth/RegistrationTest.php`:

```php
<?php

use App\Enums\RoleName;
use App\Models\User;

test('the registration screen can be rendered', function () {
    $this->get(route('register'))->assertOk();
});

test('new users register as candidates', function () {
    $response = $this->post(route('register.store'), [
        'name' => 'Amina Yusuf',
        'email' => 'amina@example.com',
        'password' => 'Password123!',
        'password_confirmation' => 'Password123!',
    ]);

    $response->assertRedirect(route('dashboard'));
    $this->assertAuthenticated();

    $user = User::whereEmail('amina@example.com')->firstOrFail();

    expect($user->hasRole(RoleName::Candidate->value))->toBeTrue()
        ->and($user->company_id)->toBeNull();
});

test('a registering user cannot inject a company id', function () {
    $this->post(route('register.store'), [
        'name' => 'Intruder',
        'email' => 'intruder@example.com',
        'password' => 'Password123!',
        'password_confirmation' => 'Password123!',
        'company_id' => 1,
    ]);

    expect(User::whereEmail('intruder@example.com')->firstOrFail()->company_id)->toBeNull();
});
```

Create `tests/Feature/Auth/CompanyRegistrationTest.php`:

```php
<?php

use App\Enums\RoleName;
use App\Models\Company;
use App\Models\User;

test('the company registration screen can be rendered', function () {
    $this->get(route('register.company'))->assertOk();
});

test('registering a company creates the company and an admin user', function () {
    $response = $this->post(route('register.company.store'), [
        'company_name' => 'Nimbus Labs',
        'name' => 'Dana Okafor',
        'email' => 'dana@nimbus.test',
        'password' => 'Password123!',
        'password_confirmation' => 'Password123!',
    ]);

    $response->assertRedirect(route('hr.jobs.index'));
    $this->assertAuthenticated();

    $company = Company::whereName('Nimbus Labs')->firstOrFail();
    $user = User::whereEmail('dana@nimbus.test')->firstOrFail();

    expect($user->company_id)->toBe($company->id)
        ->and($user->hasRole(RoleName::CompanyAdmin->value))->toBeTrue()
        ->and($company->slug)->toStartWith('nimbus-labs');
});

test('company registration requires a unique email', function () {
    User::factory()->create(['email' => 'taken@nimbus.test']);

    $this->post(route('register.company.store'), [
        'company_name' => 'Nimbus Labs',
        'name' => 'Dana Okafor',
        'email' => 'taken@nimbus.test',
        'password' => 'Password123!',
        'password_confirmation' => 'Password123!',
    ])->assertSessionHasErrors('email');

    expect(Company::count())->toBe(0);
});
```

- [ ] **Step 2: Run them to confirm they fail**

```bash
php artisan test --compact --filter="RegistrationTest|CompanyRegistrationTest"
```

Expected: FAIL — `Route [register] not defined`.

- [ ] **Step 3: Enable Fortify registration**

In `config/fortify.php`, change the `features` array to:

```php
    'features' => [
        Features::registration(),
        Features::resetPasswords(),
    ],
```

- [ ] **Step 4: Create the Fortify user creation action**

```bash
php artisan make:class Actions/Fortify/CreateNewUser --no-interaction
```

`app/Actions/Fortify/CreateNewUser.php`:

```php
<?php

namespace App\Actions\Fortify;

use App\Concerns\PasswordValidationRules;
use App\Concerns\ProfileValidationRules;
use App\Enums\RoleName;
use App\Models\User;
use Illuminate\Support\Facades\Validator;
use Laravel\Fortify\Contracts\CreatesNewUsers;

class CreateNewUser implements CreatesNewUsers
{
    use PasswordValidationRules, ProfileValidationRules;

    /**
     * Validate and create a newly registered candidate.
     *
     * @param  array<string, string>  $input
     */
    public function create(array $input): User
    {
        Validator::make($input, [
            'name' => $this->nameRules(),
            'email' => $this->emailRules(),
            'password' => $this->passwordRules(),
        ])->validate();

        $user = User::create([
            'name' => $input['name'],
            'email' => $input['email'],
            'password' => $input['password'],
        ]);

        $user->assignRole(RoleName::Candidate->value);

        return $user;
    }
}
```

Only `name`, `email` and `password` are passed to `User::create()`, so a `company_id` in the payload is discarded.

- [ ] **Step 5: Register the view and the action with Fortify**

In `app/Providers/FortifyServiceProvider.php`, add the import:

```php
use App\Actions\Fortify\CreateNewUser;
```

Extend `configureActions()`:

```php
    private function configureActions(): void
    {
        Fortify::createUsersUsing(CreateNewUser::class);
        Fortify::resetUserPasswordsUsing(ResetUserPassword::class);
    }
```

Add this inside `configureViews()`:

```php
        Fortify::registerView(fn () => Inertia::render('auth/register', [
            'passwordRules' => Password::defaults()->toPasswordRulesString(),
        ]));
```

- [ ] **Step 6: Write the company registration action**

```bash
php artisan make:class Actions/Companies/RegisterCompany --no-interaction
```

`app/Actions/Companies/RegisterCompany.php`:

```php
<?php

namespace App\Actions\Companies;

use App\Enums\RoleName;
use App\Models\Company;
use App\Models\User;
use Illuminate\Support\Facades\DB;
use Illuminate\Support\Str;

class RegisterCompany
{
    /**
     * Create a company together with its first administrator.
     */
    public function handle(string $companyName, string $name, string $email, string $password): User
    {
        return DB::transaction(function () use ($companyName, $name, $email, $password): User {
            $company = Company::create([
                'name' => $companyName,
                'slug' => $this->uniqueSlug($companyName),
            ]);

            $user = User::create([
                'name' => $name,
                'email' => $email,
                'password' => $password,
            ]);

            $user->company_id = $company->id;
            $user->save();

            $user->assignRole(RoleName::CompanyAdmin->value);

            return $user;
        });
    }

    private function uniqueSlug(string $companyName): string
    {
        $base = Str::slug($companyName) ?: 'company';

        do {
            $slug = $base.'-'.Str::lower(Str::random(6));
        } while (Company::whereSlug($slug)->exists());

        return $slug;
    }
}
```

- [ ] **Step 7: Write the form request**

```bash
php artisan make:request Auth/RegisterCompanyRequest --no-interaction
```

`app/Http/Requests/Auth/RegisterCompanyRequest.php`:

```php
<?php

namespace App\Http\Requests\Auth;

use App\Concerns\PasswordValidationRules;
use App\Concerns\ProfileValidationRules;
use Illuminate\Contracts\Validation\ValidationRule;
use Illuminate\Foundation\Http\FormRequest;

class RegisterCompanyRequest extends FormRequest
{
    use PasswordValidationRules, ProfileValidationRules;

    /**
     * @return array<string, ValidationRule|array<mixed>|string>
     */
    public function rules(): array
    {
        return [
            'company_name' => ['required', 'string', 'max:255'],
            'name' => $this->nameRules(),
            'email' => $this->emailRules(),
            'password' => $this->passwordRules(),
        ];
    }
}
```

- [ ] **Step 8: Write the controller**

```bash
php artisan make:controller Auth/RegisteredCompanyController --no-interaction
```

`app/Http/Controllers/Auth/RegisteredCompanyController.php`:

```php
<?php

namespace App\Http\Controllers\Auth;

use App\Actions\Companies\RegisterCompany;
use App\Http\Controllers\Controller;
use App\Http\Requests\Auth\RegisterCompanyRequest;
use Illuminate\Http\RedirectResponse;
use Illuminate\Support\Facades\Auth;
use Illuminate\Validation\Rules\Password;
use Inertia\Inertia;
use Inertia\Response;

class RegisteredCompanyController extends Controller
{
    /**
     * Show the company registration form.
     */
    public function create(): Response
    {
        return Inertia::render('auth/register-company', [
            'passwordRules' => Password::defaults()->toPasswordRulesString(),
        ]);
    }

    /**
     * Register a company and log its administrator in.
     */
    public function store(RegisterCompanyRequest $request, RegisterCompany $registerCompany): RedirectResponse
    {
        $user = $registerCompany->handle(
            $request->string('company_name')->toString(),
            $request->string('name')->toString(),
            $request->string('email')->toString(),
            $request->string('password')->toString(),
        );

        Auth::login($user);

        $request->session()->regenerate();

        Inertia::flash('toast', ['type' => 'success', 'message' => __('Your company workspace is ready.')]);

        return to_route('hr.jobs.index');
    }
}
```

- [ ] **Step 9: Write the dashboard redirect controller**

```bash
php artisan make:controller DashboardController --invokable --no-interaction
```

`app/Http/Controllers/DashboardController.php`:

```php
<?php

namespace App\Http\Controllers;

use App\Enums\RoleName;
use Illuminate\Http\RedirectResponse;
use Illuminate\Http\Request;

class DashboardController extends Controller
{
    /**
     * Send each user to the area that matches their role.
     */
    public function __invoke(Request $request): RedirectResponse
    {
        if ($request->user()->hasAnyRole(RoleName::hrRoles())) {
            return to_route('hr.jobs.index');
        }

        return to_route('candidate.applications.index');
    }
}
```

- [ ] **Step 10: Wire the routes**

Replace `routes/web.php` entirely:

```php
<?php

use App\Http\Controllers\Auth\RegisteredCompanyController;
use App\Http\Controllers\DashboardController;
use Illuminate\Support\Facades\Route;

Route::inertia('/', 'welcome')->name('home');

Route::middleware('guest')->group(function () {
    Route::get('register/company', [RegisteredCompanyController::class, 'create'])->name('register.company');
    Route::post('register/company', [RegisteredCompanyController::class, 'store'])->name('register.company.store');
});

Route::middleware(['auth', 'verified'])->group(function () {
    Route::get('dashboard', DashboardController::class)->name('dashboard');
});

require __DIR__.'/settings.php';
require __DIR__.'/hr.php';
```

Create `routes/hr.php` as a placeholder so the `require` resolves — Task 4 fills it in:

```php
<?php

use Illuminate\Support\Facades\Route;

Route::middleware(['auth', 'verified'])->group(function () {
    //
});
```

- [ ] **Step 11: Share roles and company with the front end**

In `app/Http/Middleware/HandleInertiaRequests.php`, replace the `auth` key inside `share()`:

```php
            'auth' => [
                'user' => $request->user(),
                'roles' => $request->user()?->getRoleNames()->all() ?? [],
                'company' => $request->user()?->company?->only(['id', 'name']),
            ],
```

Update `resources/js/types/auth.ts`:

```typescript
export type User = {
    id: number;
    name: string;
    email: string;
    avatar?: string;
    email_verified_at: string | null;
    created_at: string;
    updated_at: string;
    [key: string]: unknown;
};

export type AuthCompany = {
    id: number;
    name: string;
};

export type Auth = {
    user: User;
    roles: string[];
    company: AuthCompany | null;
};
```

- [ ] **Step 12: Build the candidate register page**

`resources/js/pages/auth/register.tsx`:

```tsx
import { Form, Head } from '@inertiajs/react';
import InputError from '@/components/input-error';
import PasswordInput from '@/components/password-input';
import TextLink from '@/components/text-link';
import { Button } from '@/components/ui/button';
import { Input } from '@/components/ui/input';
import { Label } from '@/components/ui/label';
import { Spinner } from '@/components/ui/spinner';
import { login } from '@/routes';
import { company, store } from '@/routes/register';

export default function Register() {
    return (
        <>
            <Head title="Create an account" />

            <Form
                {...store.form()}
                resetOnSuccess={['password', 'password_confirmation']}
                className="flex flex-col gap-6"
            >
                {({ processing, errors }) => (
                    <div className="grid gap-6">
                        <div className="grid gap-2">
                            <Label htmlFor="name">Name</Label>
                            <Input id="name" name="name" required autoFocus autoComplete="name" />
                            <InputError message={errors.name} />
                        </div>

                        <div className="grid gap-2">
                            <Label htmlFor="email">Email address</Label>
                            <Input id="email" type="email" name="email" required autoComplete="email" />
                            <InputError message={errors.email} />
                        </div>

                        <div className="grid gap-2">
                            <Label htmlFor="password">Password</Label>
                            <PasswordInput id="password" name="password" required autoComplete="new-password" />
                            <InputError message={errors.password} />
                        </div>

                        <div className="grid gap-2">
                            <Label htmlFor="password_confirmation">Confirm password</Label>
                            <PasswordInput
                                id="password_confirmation"
                                name="password_confirmation"
                                required
                                autoComplete="new-password"
                            />
                            <InputError message={errors.password_confirmation} />
                        </div>

                        <Button type="submit" className="mt-2 w-full" disabled={processing}>
                            {processing && <Spinner />}
                            Create account
                        </Button>

                        <div className="text-center text-sm text-muted-foreground">
                            Already registered? <TextLink href={login()}>Log in</TextLink>
                        </div>

                        <div className="text-center text-sm text-muted-foreground">
                            Hiring instead? <TextLink href={company()}>Create a company workspace</TextLink>
                        </div>
                    </div>
                )}
            </Form>
        </>
    );
}

Register.layout = {
    title: 'Create your account',
    description: 'Register to apply for open roles',
};
```

> Wayfinder derives these module paths from route **names**, so generate before you import: `php artisan wayfinder:generate --with-form --no-interaction`. Expect `register.store` → `store` in `resources/js/routes/register/index.ts`, `register.company` → `company` in the same file, and `register.company.store` → `store` in `resources/js/routes/register/company/index.ts`. If a name differs, open the generated file and use what is actually exported.

- [ ] **Step 13: Build the company register page**

`resources/js/pages/auth/register-company.tsx`:

```tsx
import { Form, Head } from '@inertiajs/react';
import InputError from '@/components/input-error';
import PasswordInput from '@/components/password-input';
import TextLink from '@/components/text-link';
import { Button } from '@/components/ui/button';
import { Input } from '@/components/ui/input';
import { Label } from '@/components/ui/label';
import { Spinner } from '@/components/ui/spinner';
import { login, register } from '@/routes';
import { store } from '@/routes/register/company';

export default function RegisterCompany() {
    return (
        <>
            <Head title="Create a company workspace" />

            <Form {...store.form()} className="flex flex-col gap-6">
                {({ processing, errors }) => (
                    <div className="grid gap-6">
                        <div className="grid gap-2">
                            <Label htmlFor="company_name">Company name</Label>
                            <Input id="company_name" name="company_name" required autoFocus />
                            <InputError message={errors.company_name} />
                        </div>

                        <div className="grid gap-2">
                            <Label htmlFor="name">Your name</Label>
                            <Input id="name" name="name" required autoComplete="name" />
                            <InputError message={errors.name} />
                        </div>

                        <div className="grid gap-2">
                            <Label htmlFor="email">Work email</Label>
                            <Input id="email" type="email" name="email" required autoComplete="email" />
                            <InputError message={errors.email} />
                        </div>

                        <div className="grid gap-2">
                            <Label htmlFor="password">Password</Label>
                            <PasswordInput id="password" name="password" required autoComplete="new-password" />
                            <InputError message={errors.password} />
                        </div>

                        <div className="grid gap-2">
                            <Label htmlFor="password_confirmation">Confirm password</Label>
                            <PasswordInput
                                id="password_confirmation"
                                name="password_confirmation"
                                required
                                autoComplete="new-password"
                            />
                            <InputError message={errors.password_confirmation} />
                        </div>

                        <Button type="submit" className="mt-2 w-full" disabled={processing}>
                            {processing && <Spinner />}
                            Create workspace
                        </Button>

                        <div className="text-center text-sm text-muted-foreground">
                            Looking for a job? <TextLink href={register()}>Register as a candidate</TextLink>
                        </div>

                        <div className="text-center text-sm text-muted-foreground">
                            Already have an account? <TextLink href={login()}>Log in</TextLink>
                        </div>
                    </div>
                )}
            </Form>
        </>
    );
}

RegisterCompany.layout = {
    title: 'Set up your hiring workspace',
    description: 'You will be the first administrator on this company',
};
```

- [ ] **Step 14: Update the dashboard test**

Replace `tests/Feature/DashboardTest.php`:

```php
<?php

use App\Enums\RoleName;
use App\Models\Company;
use App\Models\User;

test('guests are redirected to the login page', function () {
    $this->get(route('dashboard'))->assertRedirect(route('login'));
});

test('hr users land on their job postings', function () {
    $company = Company::factory()->create();
    $user = User::factory()->create(['company_id' => $company->id]);
    $user->assignRole(RoleName::Hr->value);

    $this->actingAs($user)
        ->get(route('dashboard'))
        ->assertRedirect(route('hr.jobs.index'));
});

test('candidates land on their applications', function () {
    $user = User::factory()->create();
    $user->assignRole(RoleName::Candidate->value);

    $this->actingAs($user)
        ->get(route('dashboard'))
        ->assertRedirect(route('candidate.applications.index'));
});
```

This test cannot pass until Task 3 defines `hr.jobs.index` and Task 5 defines `candidate.applications.index`. That is expected — re-run it at the end of Task 5.

- [ ] **Step 15: Run the registration tests**

```bash
php artisan test --compact --filter="RegistrationTest"
```

Expected: PASS (3 tests). `CompanyRegistrationTest` and `DashboardTest` still fail on the missing `hr.jobs.index` route; `CompanyRegistrationTest` goes green in Task 3 and `DashboardTest` in Task 5.

- [ ] **Step 16: Regenerate routes, format, commit**

```bash
php artisan wayfinder:generate --with-form --no-interaction
vendor/bin/pint --dirty --format agent
git add -A
git commit -m "feat: add candidate and company registration"
```

---

## Task 3: Job posting CRUD for HR

**Files:**
- Create: `app/Policies/JobPostingPolicy.php`, `app/Actions/JobPostings/SaveJobPosting.php`, `app/Http/Requests/Hr/JobPostingRequest.php`, `app/Http/Controllers/Hr/JobPostingController.php`, `resources/js/types/hr.ts`, `resources/js/components/job-posting-form.tsx`, `resources/js/pages/hr/jobs/{index,create,edit}.tsx`
- Modify: `routes/hr.php`, `resources/js/components/app-sidebar.tsx`
- Test: `tests/Feature/Hr/JobPostingTest.php`

**Interfaces:**
- Consumes: `JobPosting`, `JobPostingStatus`, `EmploymentType`, `PermissionName`, `RoleName` from Task 1
- Produces:
  - `SaveJobPosting::handle(User $author, array $attributes, ?JobPosting $jobPosting = null): JobPosting`
  - Route names `hr.jobs.index`, `hr.jobs.create`, `hr.jobs.store`, `hr.jobs.edit`, `hr.jobs.update`, `hr.jobs.destroy` (route parameter is `jobPosting`)
  - `JobPostingPolicy` with `viewAny`, `view`, `create`, `update`, `delete`
  - TS types `JobPostingSummary`, `JobPostingDetail`, `SelectOption` in `@/types/hr`

- [ ] **Step 1: Add the shadcn primitives you will need across the rest of the plan**

```bash
npx shadcn@latest add table textarea progress alert-dialog
```

- [ ] **Step 2: Write the failing test**

Create `tests/Feature/Hr/JobPostingTest.php`:

```php
<?php

use App\Enums\JobPostingStatus;
use App\Enums\RoleName;
use App\Models\Company;
use App\Models\JobPosting;
use App\Models\User;

function hrUser(?Company $company = null, RoleName $role = RoleName::Hr): User
{
    $company ??= Company::factory()->create();
    $user = User::factory()->create(['company_id' => $company->id]);
    $user->assignRole($role->value);

    return $user;
}

test('hr can list only their own company postings', function () {
    $user = hrUser();
    JobPosting::factory()->create(['company_id' => $user->company_id, 'created_by' => $user->id]);
    JobPosting::factory()->create();

    $this->actingAs($user)
        ->get(route('hr.jobs.index'))
        ->assertOk()
        ->assertInertia(fn ($page) => $page
            ->component('hr/jobs/index')
            ->has('jobPostings', 1)
        );
});

test('candidates cannot reach the hr area', function () {
    $candidate = User::factory()->create();
    $candidate->assignRole(RoleName::Candidate->value);

    $this->actingAs($candidate)
        ->get(route('hr.jobs.index'))
        ->assertForbidden();
});

test('hr can create a posting scoped to their company', function () {
    $user = hrUser();

    $this->actingAs($user)->post(route('hr.jobs.store'), [
        'title' => 'Backend Engineer',
        'description' => 'Build and maintain our billing services.',
        'requirements' => 'PHP, Laravel, SQL, queues.',
        'employment_type' => 'full_time',
        'location' => 'Riyadh',
        'min_years_experience' => 3,
        'status' => 'published',
    ])->assertRedirect(route('hr.jobs.index'));

    $posting = JobPosting::whereTitle('Backend Engineer')->firstOrFail();

    expect($posting->company_id)->toBe($user->company_id)
        ->and($posting->created_by)->toBe($user->id)
        ->and($posting->status)->toBe(JobPostingStatus::Published)
        ->and($posting->published_at)->not->toBeNull();
});

test('a draft posting has no published date', function () {
    $user = hrUser();

    $this->actingAs($user)->post(route('hr.jobs.store'), [
        'title' => 'Draft Role',
        'description' => 'Not live yet.',
        'requirements' => 'None yet.',
        'employment_type' => 'contract',
        'location' => 'Remote',
        'min_years_experience' => 0,
        'status' => 'draft',
    ]);

    expect(JobPosting::whereTitle('Draft Role')->firstOrFail()->published_at)->toBeNull();
});

test('hr cannot edit a posting from another company', function () {
    $user = hrUser();
    $foreign = JobPosting::factory()->create();

    $this->actingAs($user)
        ->get(route('hr.jobs.edit', $foreign))
        ->assertForbidden();

    $this->actingAs($user)
        ->put(route('hr.jobs.update', $foreign), [
            'title' => 'Hijacked',
            'description' => 'x',
            'requirements' => 'y',
            'employment_type' => 'full_time',
            'location' => 'Nowhere',
            'min_years_experience' => 0,
            'status' => 'draft',
        ])
        ->assertForbidden();

    expect($foreign->fresh()->title)->not->toBe('Hijacked');
});

test('hr can update and delete their own posting', function () {
    $user = hrUser();
    $posting = JobPosting::factory()->create([
        'company_id' => $user->company_id,
        'created_by' => $user->id,
    ]);

    $this->actingAs($user)->put(route('hr.jobs.update', $posting), [
        'title' => 'Updated Title',
        'description' => 'Updated description.',
        'requirements' => 'Updated requirements.',
        'employment_type' => 'part_time',
        'location' => 'Jeddah',
        'min_years_experience' => 2,
        'status' => 'published',
    ])->assertRedirect(route('hr.jobs.index'));

    expect($posting->fresh()->title)->toBe('Updated Title');

    $this->actingAs($user)->delete(route('hr.jobs.destroy', $posting));

    expect(JobPosting::find($posting->id))->toBeNull();
});

test('validation rejects an empty title', function () {
    $user = hrUser();

    $this->actingAs($user)->post(route('hr.jobs.store'), [
        'title' => '',
        'description' => 'x',
        'requirements' => 'y',
        'employment_type' => 'full_time',
        'location' => 'Riyadh',
        'min_years_experience' => 0,
        'status' => 'draft',
    ])->assertSessionHasErrors('title');
});
```

- [ ] **Step 3: Run it to confirm it fails**

```bash
php artisan test --compact --filter=JobPostingTest
```

Expected: FAIL — `Route [hr.jobs.index] not defined`.

- [ ] **Step 4: Write the policy**

```bash
php artisan make:policy JobPostingPolicy --model=JobPosting --no-interaction
```

Replace `app/Policies/JobPostingPolicy.php`:

```php
<?php

namespace App\Policies;

use App\Enums\PermissionName;
use App\Models\JobPosting;
use App\Models\User;

class JobPostingPolicy
{
    public function viewAny(User $user): bool
    {
        return $user->can(PermissionName::ManageJobs->value);
    }

    public function create(User $user): bool
    {
        return $user->can(PermissionName::ManageJobs->value) && $user->company_id !== null;
    }

    public function view(User $user, JobPosting $jobPosting): bool
    {
        return $this->belongsToUsersCompany($user, $jobPosting);
    }

    public function update(User $user, JobPosting $jobPosting): bool
    {
        return $this->belongsToUsersCompany($user, $jobPosting);
    }

    public function delete(User $user, JobPosting $jobPosting): bool
    {
        return $this->belongsToUsersCompany($user, $jobPosting);
    }

    /**
     * The permission check alone is not enough: without the company comparison
     * any HR user could read or edit another company's postings.
     */
    private function belongsToUsersCompany(User $user, JobPosting $jobPosting): bool
    {
        return $user->can(PermissionName::ManageJobs->value)
            && $user->company_id !== null
            && $user->company_id === $jobPosting->company_id;
    }
}
```

Laravel discovers `App\Policies\JobPostingPolicy` for `App\Models\JobPosting` automatically. No registration needed.

- [ ] **Step 5: Write the action**

```bash
php artisan make:class Actions/JobPostings/SaveJobPosting --no-interaction
```

`app/Actions/JobPostings/SaveJobPosting.php`:

```php
<?php

namespace App\Actions\JobPostings;

use App\Enums\JobPostingStatus;
use App\Models\JobPosting;
use App\Models\User;

class SaveJobPosting
{
    /**
     * Create or update a posting, stamping ownership and publish date.
     *
     * @param  array<string, mixed>  $attributes
     */
    public function handle(User $author, array $attributes, ?JobPosting $jobPosting = null): JobPosting
    {
        $jobPosting ??= new JobPosting;

        $jobPosting->fill($attributes);

        if (! $jobPosting->exists) {
            $jobPosting->company_id = $author->company_id;
            $jobPosting->created_by = $author->id;
        }

        if ($jobPosting->status === JobPostingStatus::Published && $jobPosting->published_at === null) {
            $jobPosting->published_at = now();
        }

        $jobPosting->save();

        return $jobPosting;
    }
}
```

- [ ] **Step 6: Write the form request**

```bash
php artisan make:request Hr/JobPostingRequest --no-interaction
```

`app/Http/Requests/Hr/JobPostingRequest.php`:

```php
<?php

namespace App\Http\Requests\Hr;

use App\Enums\EmploymentType;
use App\Enums\JobPostingStatus;
use Illuminate\Contracts\Validation\ValidationRule;
use Illuminate\Foundation\Http\FormRequest;
use Illuminate\Validation\Rule;

class JobPostingRequest extends FormRequest
{
    /**
     * @return array<string, ValidationRule|array<mixed>|string>
     */
    public function rules(): array
    {
        return [
            'title' => ['required', 'string', 'max:255'],
            'description' => ['required', 'string', 'max:10000'],
            'requirements' => ['required', 'string', 'max:10000'],
            'employment_type' => ['required', Rule::enum(EmploymentType::class)],
            'location' => ['required', 'string', 'max:255'],
            'min_years_experience' => ['required', 'integer', 'min:0', 'max:50'],
            'status' => ['required', Rule::enum(JobPostingStatus::class)],
        ];
    }
}
```

- [ ] **Step 7: Write the controller**

```bash
php artisan make:controller Hr/JobPostingController --no-interaction
```

`app/Http/Controllers/Hr/JobPostingController.php`:

```php
<?php

namespace App\Http\Controllers\Hr;

use App\Actions\JobPostings\SaveJobPosting;
use App\Enums\EmploymentType;
use App\Enums\JobPostingStatus;
use App\Http\Controllers\Controller;
use App\Http\Requests\Hr\JobPostingRequest;
use App\Models\JobPosting;
use Illuminate\Http\RedirectResponse;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Gate;
use Inertia\Inertia;
use Inertia\Response;

class JobPostingController extends Controller
{
    /**
     * List the postings belonging to the current user's company.
     */
    public function index(Request $request): Response
    {
        Gate::authorize('viewAny', JobPosting::class);

        $jobPostings = JobPosting::query()
            ->where('company_id', $request->user()->company_id)
            ->withCount('applications')
            ->latest()
            ->get()
            ->map(fn (JobPosting $jobPosting): array => [
                'id' => $jobPosting->id,
                'title' => $jobPosting->title,
                'location' => $jobPosting->location,
                'employment_type' => $jobPosting->employment_type->label(),
                'status' => $jobPosting->status->value,
                'status_label' => $jobPosting->status->label(),
                'applications_count' => $jobPosting->applications_count,
                'created_at' => $jobPosting->created_at?->toDateString(),
            ]);

        return Inertia::render('hr/jobs/index', [
            'jobPostings' => $jobPostings,
        ]);
    }

    /**
     * Show the create form.
     */
    public function create(): Response
    {
        Gate::authorize('create', JobPosting::class);

        return Inertia::render('hr/jobs/create', [
            'employmentTypes' => $this->employmentTypes(),
            'statuses' => $this->statuses(),
        ]);
    }

    /**
     * Persist a new posting.
     */
    public function store(JobPostingRequest $request, SaveJobPosting $saveJobPosting): RedirectResponse
    {
        Gate::authorize('create', JobPosting::class);

        $saveJobPosting->handle($request->user(), $request->validated());

        Inertia::flash('toast', ['type' => 'success', 'message' => __('Job posting created.')]);

        return to_route('hr.jobs.index');
    }

    /**
     * Show the edit form.
     */
    public function edit(JobPosting $jobPosting): Response
    {
        Gate::authorize('update', $jobPosting);

        return Inertia::render('hr/jobs/edit', [
            'jobPosting' => [
                'id' => $jobPosting->id,
                'title' => $jobPosting->title,
                'description' => $jobPosting->description,
                'requirements' => $jobPosting->requirements,
                'employment_type' => $jobPosting->employment_type->value,
                'location' => $jobPosting->location,
                'min_years_experience' => $jobPosting->min_years_experience,
                'status' => $jobPosting->status->value,
            ],
            'employmentTypes' => $this->employmentTypes(),
            'statuses' => $this->statuses(),
        ]);
    }

    /**
     * Update an existing posting.
     */
    public function update(JobPostingRequest $request, JobPosting $jobPosting, SaveJobPosting $saveJobPosting): RedirectResponse
    {
        Gate::authorize('update', $jobPosting);

        $saveJobPosting->handle($request->user(), $request->validated(), $jobPosting);

        Inertia::flash('toast', ['type' => 'success', 'message' => __('Job posting updated.')]);

        return to_route('hr.jobs.index');
    }

    /**
     * Delete a posting and everything attached to it.
     */
    public function destroy(JobPosting $jobPosting): RedirectResponse
    {
        Gate::authorize('delete', $jobPosting);

        $jobPosting->delete();

        Inertia::flash('toast', ['type' => 'success', 'message' => __('Job posting deleted.')]);

        return to_route('hr.jobs.index');
    }

    /**
     * @return array<int, array{value: string, label: string}>
     */
    private function employmentTypes(): array
    {
        return array_map(
            fn (EmploymentType $type): array => ['value' => $type->value, 'label' => $type->label()],
            EmploymentType::cases(),
        );
    }

    /**
     * @return array<int, array{value: string, label: string}>
     */
    private function statuses(): array
    {
        return array_map(
            fn (JobPostingStatus $status): array => ['value' => $status->value, 'label' => $status->label()],
            JobPostingStatus::cases(),
        );
    }
}
```

- [ ] **Step 8: Write the HR routes**

Replace `routes/hr.php`:

```php
<?php

use App\Enums\RoleName;
use App\Http\Controllers\Hr\JobPostingController;
use Illuminate\Support\Facades\Route;

Route::middleware(['auth', 'verified', 'role:'.implode('|', RoleName::hrRoles())])
    ->prefix('hr')
    ->name('hr.')
    ->group(function () {
        Route::resource('jobs', JobPostingController::class)
            ->parameters(['jobs' => 'jobPosting'])
            ->except(['show']);
    });
```

Spatie's `role:` middleware aborts with 403 for a candidate, which is what the second test asserts.

- [ ] **Step 9: Add the shared TypeScript types**

`resources/js/types/hr.ts`:

```typescript
export type SelectOption = {
    value: string;
    label: string;
};

export type JobPostingSummary = {
    id: number;
    title: string;
    location: string;
    employment_type: string;
    status: string;
    status_label: string;
    applications_count: number;
    created_at: string | null;
};

export type JobPostingFormValues = {
    id?: number;
    title: string;
    description: string;
    requirements: string;
    employment_type: string;
    location: string;
    min_years_experience: number;
    status: string;
};
```

Re-export it from `resources/js/types/index.ts`:

```typescript
export type * from './auth';
export type * from './hr';
export type * from './navigation';
export type * from './ui';
```

- [ ] **Step 10: Build the shared posting form**

Radix's `<Select>` renders no native form control, so it submits nothing inside Inertia's uncontrolled `<Form>`. Use a native `<select>` here.

`resources/js/components/job-posting-form.tsx`:

```tsx
import { Form } from '@inertiajs/react';
import InputError from '@/components/input-error';
import { Button } from '@/components/ui/button';
import { Input } from '@/components/ui/input';
import { Label } from '@/components/ui/label';
import { Spinner } from '@/components/ui/spinner';
import { Textarea } from '@/components/ui/textarea';
import type { JobPostingFormValues, SelectOption } from '@/types/hr';

type Props = {
    action: { action: string; method: 'post' | 'put' };
    submitLabel: string;
    employmentTypes: SelectOption[];
    statuses: SelectOption[];
    jobPosting?: JobPostingFormValues;
};

const selectClass =
    'h-9 w-full rounded-md border border-input bg-transparent px-3 py-1 text-sm shadow-xs outline-none focus-visible:ring-[3px] focus-visible:ring-ring/50';

export default function JobPostingForm({
    action,
    submitLabel,
    employmentTypes,
    statuses,
    jobPosting,
}: Props) {
    return (
        <Form {...action} className="flex max-w-3xl flex-col gap-6">
            {({ processing, errors }) => (
                <>
                    <div className="grid gap-2">
                        <Label htmlFor="title">Title</Label>
                        <Input id="title" name="title" defaultValue={jobPosting?.title} required autoFocus />
                        <InputError message={errors.title} />
                    </div>

                    <div className="grid gap-2">
                        <Label htmlFor="description">Description</Label>
                        <Textarea
                            id="description"
                            name="description"
                            rows={6}
                            defaultValue={jobPosting?.description}
                            required
                        />
                        <InputError message={errors.description} />
                    </div>

                    <div className="grid gap-2">
                        <Label htmlFor="requirements">Requirements</Label>
                        <Textarea
                            id="requirements"
                            name="requirements"
                            rows={5}
                            defaultValue={jobPosting?.requirements}
                            placeholder="One requirement per line. The AI screens CVs against this."
                            required
                        />
                        <InputError message={errors.requirements} />
                    </div>

                    <div className="grid gap-4 sm:grid-cols-2">
                        <div className="grid gap-2">
                            <Label htmlFor="employment_type">Employment type</Label>
                            <select
                                id="employment_type"
                                name="employment_type"
                                className={selectClass}
                                defaultValue={jobPosting?.employment_type ?? employmentTypes[0]?.value}
                            >
                                {employmentTypes.map((option) => (
                                    <option key={option.value} value={option.value}>
                                        {option.label}
                                    </option>
                                ))}
                            </select>
                            <InputError message={errors.employment_type} />
                        </div>

                        <div className="grid gap-2">
                            <Label htmlFor="location">Location</Label>
                            <Input id="location" name="location" defaultValue={jobPosting?.location} required />
                            <InputError message={errors.location} />
                        </div>

                        <div className="grid gap-2">
                            <Label htmlFor="min_years_experience">Minimum years of experience</Label>
                            <Input
                                id="min_years_experience"
                                name="min_years_experience"
                                type="number"
                                min={0}
                                max={50}
                                defaultValue={jobPosting?.min_years_experience ?? 0}
                                required
                            />
                            <InputError message={errors.min_years_experience} />
                        </div>

                        <div className="grid gap-2">
                            <Label htmlFor="status">Status</Label>
                            <select
                                id="status"
                                name="status"
                                className={selectClass}
                                defaultValue={jobPosting?.status ?? 'draft'}
                            >
                                {statuses.map((option) => (
                                    <option key={option.value} value={option.value}>
                                        {option.label}
                                    </option>
                                ))}
                            </select>
                            <InputError message={errors.status} />
                        </div>
                    </div>

                    <div>
                        <Button type="submit" disabled={processing}>
                            {processing && <Spinner />}
                            {submitLabel}
                        </Button>
                    </div>
                </>
            )}
        </Form>
    );
}
```

- [ ] **Step 11: Build the three HR job pages**

`resources/js/pages/hr/jobs/index.tsx`:

```tsx
import { Head, Link } from '@inertiajs/react';
import Heading from '@/components/heading';
import { Badge } from '@/components/ui/badge';
import { Button } from '@/components/ui/button';
import {
    Table,
    TableBody,
    TableCell,
    TableHead,
    TableHeader,
    TableRow,
} from '@/components/ui/table';
import { applications } from '@/routes/hr/jobs';
import { create, edit } from '@/routes/hr/jobs';
import type { JobPostingSummary } from '@/types/hr';

type Props = {
    jobPostings: JobPostingSummary[];
};

export default function HrJobsIndex({ jobPostings }: Props) {
    return (
        <div className="flex flex-col gap-6 p-4">
            <Head title="Job postings" />

            <div className="flex items-center justify-between">
                <Heading title="Job postings" description="Roles your company is hiring for" />
                <Button asChild>
                    <Link href={create()}>New posting</Link>
                </Button>
            </div>

            {jobPostings.length === 0 ? (
                <p className="text-sm text-muted-foreground">
                    No postings yet. Create one to start receiving applications.
                </p>
            ) : (
                <Table>
                    <TableHeader>
                        <TableRow>
                            <TableHead>Title</TableHead>
                            <TableHead>Location</TableHead>
                            <TableHead>Type</TableHead>
                            <TableHead>Status</TableHead>
                            <TableHead className="text-right">Candidates</TableHead>
                            <TableHead className="text-right">Actions</TableHead>
                        </TableRow>
                    </TableHeader>
                    <TableBody>
                        {jobPostings.map((posting) => (
                            <TableRow key={posting.id}>
                                <TableCell className="font-medium">{posting.title}</TableCell>
                                <TableCell>{posting.location}</TableCell>
                                <TableCell>{posting.employment_type}</TableCell>
                                <TableCell>
                                    <Badge variant={posting.status === 'published' ? 'default' : 'secondary'}>
                                        {posting.status_label}
                                    </Badge>
                                </TableCell>
                                <TableCell className="text-right">{posting.applications_count}</TableCell>
                                <TableCell className="space-x-2 text-right">
                                    <Button variant="ghost" size="sm" asChild>
                                        <Link href={applications(posting.id)}>Candidates</Link>
                                    </Button>
                                    <Button variant="ghost" size="sm" asChild>
                                        <Link href={edit(posting.id)}>Edit</Link>
                                    </Button>
                                </TableCell>
                            </TableRow>
                        ))}
                    </TableBody>
                </Table>
            )}
        </div>
    );
}
```

> The `applications` import does not exist until Task 7. Until then, delete that `<Button>` block and the `applications` import, or the page will fail to compile.

`resources/js/pages/hr/jobs/create.tsx`:

```tsx
import { Head } from '@inertiajs/react';
import Heading from '@/components/heading';
import JobPostingForm from '@/components/job-posting-form';
import { store } from '@/routes/hr/jobs';
import type { SelectOption } from '@/types/hr';

type Props = {
    employmentTypes: SelectOption[];
    statuses: SelectOption[];
};

export default function HrJobsCreate({ employmentTypes, statuses }: Props) {
    return (
        <div className="flex flex-col gap-6 p-4">
            <Head title="New job posting" />
            <Heading title="New job posting" description="Publish it now or save it as a draft" />

            <JobPostingForm
                action={store.form()}
                submitLabel="Create posting"
                employmentTypes={employmentTypes}
                statuses={statuses}
            />
        </div>
    );
}
```

`resources/js/pages/hr/jobs/edit.tsx`:

```tsx
import { Head } from '@inertiajs/react';
import Heading from '@/components/heading';
import JobPostingForm from '@/components/job-posting-form';
import { update } from '@/routes/hr/jobs';
import type { JobPostingFormValues, SelectOption } from '@/types/hr';

type Props = {
    jobPosting: JobPostingFormValues;
    employmentTypes: SelectOption[];
    statuses: SelectOption[];
};

export default function HrJobsEdit({ jobPosting, employmentTypes, statuses }: Props) {
    return (
        <div className="flex flex-col gap-6 p-4">
            <Head title={`Edit ${jobPosting.title}`} />
            <Heading title="Edit job posting" description={jobPosting.title} />

            <JobPostingForm
                action={update.form(jobPosting.id!)}
                submitLabel="Save changes"
                employmentTypes={employmentTypes}
                statuses={statuses}
                jobPosting={jobPosting}
            />
        </div>
    );
}
```

- [ ] **Step 12: Make the sidebar role-aware**

Replace the top of `resources/js/components/app-sidebar.tsx` — imports, `mainNavItems`, and the `<NavMain>` call:

```tsx
import { Link, usePage } from '@inertiajs/react';
import { Briefcase, FileText, Search, Users } from 'lucide-react';
import AppLogo from '@/components/app-logo';
import { NavFooter } from '@/components/nav-footer';
import { NavMain } from '@/components/nav-main';
import { NavUser } from '@/components/nav-user';
import {
    Sidebar,
    SidebarContent,
    SidebarFooter,
    SidebarHeader,
    SidebarMenu,
    SidebarMenuButton,
    SidebarMenuItem,
} from '@/components/ui/sidebar';
import { dashboard } from '@/routes';
import { index as candidateApplications } from '@/routes/my/applications';
import { index as hrJobs } from '@/routes/hr/jobs';
import { index as hrTeam } from '@/routes/hr/team';
import { index as jobBoard } from '@/routes/jobs';
import type { NavItem } from '@/types';

const hrNavItems: NavItem[] = [
    { title: 'Job postings', href: hrJobs(), icon: Briefcase },
    { title: 'Team', href: hrTeam(), icon: Users },
];

const candidateNavItems: NavItem[] = [
    { title: 'Browse jobs', href: jobBoard(), icon: Search },
    { title: 'My applications', href: candidateApplications(), icon: FileText },
];

export function AppSidebar() {
    const { auth } = usePage().props;
    const isHr = auth.roles.includes('company_admin') || auth.roles.includes('hr');
    const isAdmin = auth.roles.includes('company_admin');

    const navItems = isHr
        ? hrNavItems.filter((item) => isAdmin || item.title !== 'Team')
        : candidateNavItems;

    return (
        <Sidebar collapsible="icon" variant="inset">
            <SidebarHeader>
                <SidebarMenu>
                    <SidebarMenuItem>
                        <SidebarMenuButton size="lg" asChild>
                            <Link href={dashboard()} prefetch>
                                <AppLogo />
                            </Link>
                        </SidebarMenuButton>
                    </SidebarMenuItem>
                </SidebarMenu>
            </SidebarHeader>

            <SidebarContent>
                <NavMain items={navItems} />
            </SidebarContent>

            <SidebarFooter>
                <NavUser />
            </SidebarFooter>
        </Sidebar>
    );
}
```

Delete the now-unused `footerNavItems` constant and the `BookOpen`, `FolderGit2`, `LayoutGrid` imports.

> Three of these route imports (`my/applications`, `hr/team`, `jobs`) do not exist until Tasks 4 and 5. Comment out the corresponding nav entries until then, or do this step last.

- [ ] **Step 13: Regenerate routes and run the tests**

```bash
php artisan wayfinder:generate --with-form --no-interaction
php artisan test --compact --filter="JobPostingTest|CompanyRegistrationTest"
```

Expected: PASS — 7 job posting tests plus 3 company registration tests.

- [ ] **Step 14: Format and commit**

```bash
vendor/bin/pint --dirty --format agent
git add -A
git commit -m "feat: add hr job posting crud with company-scoped policy"
```

---

## Task 4: Team invitations

**Files:**
- Create: `app/Policies/CompanyPolicy.php`, `app/Actions/Companies/{InviteTeamMember,AcceptInvitation}.php`, `app/Http/Requests/Hr/InviteTeamMemberRequest.php`, `app/Http/Requests/Auth/AcceptInvitationRequest.php`, `app/Http/Controllers/Hr/TeamController.php`, `app/Http/Controllers/Auth/InvitationController.php`, `resources/js/pages/hr/team.tsx`, `resources/js/pages/auth/accept-invitation.tsx`
- Modify: `routes/hr.php`, `routes/web.php`
- Test: `tests/Feature/Hr/TeamInvitationTest.php`

**Interfaces:**
- Consumes: `Company`, `CompanyInvitation`, `RoleName`, `PermissionName` from Task 1
- Produces:
  - `InviteTeamMember::handle(User $inviter, string $email): CompanyInvitation`
  - `AcceptInvitation::handle(CompanyInvitation $invitation, string $name, string $password): User`
  - Route names `hr.team.index`, `hr.team.invitations.store`, `hr.team.invitations.destroy`, `invitations.show`, `invitations.store`
  - `CompanyPolicy::manageTeam(User, Company): bool`

- [ ] **Step 1: Write the failing test**

Create `tests/Feature/Hr/TeamInvitationTest.php`:

```php
<?php

use App\Enums\RoleName;
use App\Models\Company;
use App\Models\CompanyInvitation;
use App\Models\User;

function companyAdmin(?Company $company = null): User
{
    $company ??= Company::factory()->create();
    $user = User::factory()->create(['company_id' => $company->id]);
    $user->assignRole(RoleName::CompanyAdmin->value);

    return $user;
}

test('an admin can invite a team member', function () {
    $admin = companyAdmin();

    $this->actingAs($admin)
        ->post(route('hr.team.invitations.store'), ['email' => 'new.hr@example.test'])
        ->assertRedirect(route('hr.team.index'));

    $invitation = CompanyInvitation::whereEmail('new.hr@example.test')->firstOrFail();

    expect($invitation->company_id)->toBe($admin->company_id)
        ->and($invitation->invited_by)->toBe($admin->id)
        ->and(strlen($invitation->token))->toBe(64)
        ->and($invitation->isPending())->toBeTrue();
});

test('a plain hr user cannot invite', function () {
    $company = Company::factory()->create();
    $hr = User::factory()->create(['company_id' => $company->id]);
    $hr->assignRole(RoleName::Hr->value);

    $this->actingAs($hr)
        ->post(route('hr.team.invitations.store'), ['email' => 'nope@example.test'])
        ->assertForbidden();
});

test('inviting the same email twice fails validation', function () {
    $admin = companyAdmin();
    CompanyInvitation::factory()->create([
        'company_id' => $admin->company_id,
        'invited_by' => $admin->id,
        'email' => 'dupe@example.test',
    ]);

    $this->actingAs($admin)
        ->post(route('hr.team.invitations.store'), ['email' => 'dupe@example.test'])
        ->assertSessionHasErrors('email');
});

test('accepting an invitation creates an hr user on that company', function () {
    $admin = companyAdmin();
    $invitation = CompanyInvitation::factory()->create([
        'company_id' => $admin->company_id,
        'invited_by' => $admin->id,
        'email' => 'joiner@example.test',
    ]);

    $this->get(route('invitations.show', $invitation->token))->assertOk();

    $this->post(route('invitations.store', $invitation->token), [
        'name' => 'Joiner Person',
        'password' => 'Password123!',
        'password_confirmation' => 'Password123!',
    ])->assertRedirect(route('hr.jobs.index'));

    $user = User::whereEmail('joiner@example.test')->firstOrFail();

    expect($user->company_id)->toBe($admin->company_id)
        ->and($user->hasRole(RoleName::Hr->value))->toBeTrue()
        ->and($invitation->fresh()->accepted_at)->not->toBeNull();
});

test('an expired invitation cannot be accepted', function () {
    $admin = companyAdmin();
    $invitation = CompanyInvitation::factory()->expired()->create([
        'company_id' => $admin->company_id,
        'invited_by' => $admin->id,
    ]);

    $this->get(route('invitations.show', $invitation->token))->assertNotFound();
});

test('an invitation cannot be used twice', function () {
    $admin = companyAdmin();
    $invitation = CompanyInvitation::factory()->accepted()->create([
        'company_id' => $admin->company_id,
        'invited_by' => $admin->id,
    ]);

    $this->get(route('invitations.show', $invitation->token))->assertNotFound();
});

test('an admin sees their own team only', function () {
    $admin = companyAdmin();
    User::factory()->create(['company_id' => $admin->company_id]);
    User::factory()->create();

    $this->actingAs($admin)
        ->get(route('hr.team.index'))
        ->assertOk()
        ->assertInertia(fn ($page) => $page->component('hr/team')->has('members', 2));
});
```

- [ ] **Step 2: Run it to confirm it fails**

```bash
php artisan test --compact --filter=TeamInvitationTest
```

Expected: FAIL — `Route [hr.team.invitations.store] not defined`.

- [ ] **Step 3: Write the company policy**

```bash
php artisan make:policy CompanyPolicy --model=Company --no-interaction
```

Replace `app/Policies/CompanyPolicy.php`:

```php
<?php

namespace App\Policies;

use App\Enums\PermissionName;
use App\Models\Company;
use App\Models\User;

class CompanyPolicy
{
    public function manageTeam(User $user, Company $company): bool
    {
        return $user->can(PermissionName::ManageTeam->value)
            && $user->company_id !== null
            && $user->company_id === $company->id;
    }
}
```

- [ ] **Step 4: Write the two actions**

```bash
php artisan make:class Actions/Companies/InviteTeamMember --no-interaction
php artisan make:class Actions/Companies/AcceptInvitation --no-interaction
```

`app/Actions/Companies/InviteTeamMember.php`:

```php
<?php

namespace App\Actions\Companies;

use App\Models\CompanyInvitation;
use App\Models\User;
use Illuminate\Support\Str;

class InviteTeamMember
{
    /**
     * Create a single-use invitation for an HR seat on the inviter's company.
     */
    public function handle(User $inviter, string $email): CompanyInvitation
    {
        return CompanyInvitation::create([
            'company_id' => $inviter->company_id,
            'invited_by' => $inviter->id,
            'email' => $email,
            'token' => Str::random(64),
            'expires_at' => now()->addDays(config('hr.invitations.expires_after_days')),
        ]);
    }
}
```

`app/Actions/Companies/AcceptInvitation.php`:

```php
<?php

namespace App\Actions\Companies;

use App\Enums\RoleName;
use App\Models\CompanyInvitation;
use App\Models\User;
use Illuminate\Support\Facades\DB;

class AcceptInvitation
{
    /**
     * Turn a pending invitation into an HR user on the inviting company.
     */
    public function handle(CompanyInvitation $invitation, string $name, string $password): User
    {
        return DB::transaction(function () use ($invitation, $name, $password): User {
            $user = User::create([
                'name' => $name,
                'email' => $invitation->email,
                'password' => $password,
            ]);

            $user->company_id = $invitation->company_id;
            $user->save();

            $user->assignRole(RoleName::Hr->value);

            $invitation->forceFill(['accepted_at' => now()])->save();

            return $user;
        });
    }
}
```

- [ ] **Step 5: Write the form requests**

```bash
php artisan make:request Hr/InviteTeamMemberRequest --no-interaction
php artisan make:request Auth/AcceptInvitationRequest --no-interaction
```

`app/Http/Requests/Hr/InviteTeamMemberRequest.php`:

```php
<?php

namespace App\Http\Requests\Hr;

use App\Models\CompanyInvitation;
use App\Models\User;
use Illuminate\Contracts\Validation\ValidationRule;
use Illuminate\Foundation\Http\FormRequest;
use Illuminate\Validation\Rule;

class InviteTeamMemberRequest extends FormRequest
{
    /**
     * @return array<string, ValidationRule|array<mixed>|string>
     */
    public function rules(): array
    {
        return [
            'email' => [
                'required',
                'string',
                'email',
                'max:255',
                Rule::unique(User::class, 'email'),
                Rule::unique(CompanyInvitation::class, 'email')
                    ->where('company_id', $this->user()->company_id)
                    ->whereNull('accepted_at'),
            ],
        ];
    }

    /**
     * @return array<string, string>
     */
    public function messages(): array
    {
        return [
            'email.unique' => __('That email already has an account or a pending invitation.'),
        ];
    }
}
```

`app/Http/Requests/Auth/AcceptInvitationRequest.php`:

```php
<?php

namespace App\Http\Requests\Auth;

use App\Concerns\PasswordValidationRules;
use App\Concerns\ProfileValidationRules;
use Illuminate\Contracts\Validation\ValidationRule;
use Illuminate\Foundation\Http\FormRequest;

class AcceptInvitationRequest extends FormRequest
{
    use PasswordValidationRules, ProfileValidationRules;

    /**
     * @return array<string, ValidationRule|array<mixed>|string>
     */
    public function rules(): array
    {
        return [
            'name' => $this->nameRules(),
            'password' => $this->passwordRules(),
        ];
    }
}
```

The email is not accepted from the request — it comes from the invitation record, so an invitee cannot claim a different address.

- [ ] **Step 6: Write the controllers**

```bash
php artisan make:controller Hr/TeamController --no-interaction
php artisan make:controller Auth/InvitationController --no-interaction
```

`app/Http/Controllers/Hr/TeamController.php`:

```php
<?php

namespace App\Http\Controllers\Hr;

use App\Actions\Companies\InviteTeamMember;
use App\Http\Controllers\Controller;
use App\Http\Requests\Hr\InviteTeamMemberRequest;
use App\Models\CompanyInvitation;
use App\Models\User;
use Illuminate\Http\RedirectResponse;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Gate;
use Inertia\Inertia;
use Inertia\Response;

class TeamController extends Controller
{
    /**
     * Show the company's members and pending invitations.
     */
    public function index(Request $request): Response
    {
        Gate::authorize('manageTeam', $request->user()->company);

        $members = User::query()
            ->where('company_id', $request->user()->company_id)
            ->with('roles')
            ->orderBy('name')
            ->get()
            ->map(fn (User $user): array => [
                'id' => $user->id,
                'name' => $user->name,
                'email' => $user->email,
                'roles' => $user->getRoleNames()->all(),
            ]);

        $invitations = CompanyInvitation::query()
            ->where('company_id', $request->user()->company_id)
            ->whereNull('accepted_at')
            ->latest()
            ->get()
            ->map(fn (CompanyInvitation $invitation): array => [
                'id' => $invitation->id,
                'email' => $invitation->email,
                'expires_at' => $invitation->expires_at->toDateString(),
                'expired' => $invitation->expires_at->isPast(),
                'url' => route('invitations.show', $invitation->token),
            ]);

        return Inertia::render('hr/team', [
            'members' => $members,
            'invitations' => $invitations,
        ]);
    }

    /**
     * Invite a new HR user.
     */
    public function store(InviteTeamMemberRequest $request, InviteTeamMember $inviteTeamMember): RedirectResponse
    {
        Gate::authorize('manageTeam', $request->user()->company);

        $inviteTeamMember->handle($request->user(), $request->string('email')->toString());

        Inertia::flash('toast', ['type' => 'success', 'message' => __('Invitation created. Share the link with them.')]);

        return to_route('hr.team.index');
    }

    /**
     * Revoke a pending invitation.
     */
    public function destroy(Request $request, CompanyInvitation $invitation): RedirectResponse
    {
        Gate::authorize('manageTeam', $request->user()->company);

        abort_unless($invitation->company_id === $request->user()->company_id, 403);

        $invitation->delete();

        Inertia::flash('toast', ['type' => 'success', 'message' => __('Invitation revoked.')]);

        return to_route('hr.team.index');
    }
}
```

`app/Http/Controllers/Auth/InvitationController.php`:

```php
<?php

namespace App\Http\Controllers\Auth;

use App\Actions\Companies\AcceptInvitation;
use App\Http\Controllers\Controller;
use App\Http\Requests\Auth\AcceptInvitationRequest;
use App\Models\CompanyInvitation;
use Illuminate\Http\RedirectResponse;
use Illuminate\Support\Facades\Auth;
use Illuminate\Validation\Rules\Password;
use Inertia\Inertia;
use Inertia\Response;

class InvitationController extends Controller
{
    /**
     * Show the invitation acceptance form.
     */
    public function show(string $token): Response
    {
        $invitation = $this->pendingInvitation($token);

        return Inertia::render('auth/accept-invitation', [
            'token' => $token,
            'email' => $invitation->email,
            'companyName' => $invitation->company->name,
            'passwordRules' => Password::defaults()->toPasswordRulesString(),
        ]);
    }

    /**
     * Accept the invitation and log the new HR user in.
     */
    public function store(AcceptInvitationRequest $request, string $token, AcceptInvitation $acceptInvitation): RedirectResponse
    {
        $invitation = $this->pendingInvitation($token);

        $user = $acceptInvitation->handle(
            $invitation,
            $request->string('name')->toString(),
            $request->string('password')->toString(),
        );

        Auth::login($user);

        $request->session()->regenerate();

        Inertia::flash('toast', ['type' => 'success', 'message' => __('Welcome to the team.')]);

        return to_route('hr.jobs.index');
    }

    /**
     * Resolve a token to an invitation that is neither expired nor already used.
     */
    private function pendingInvitation(string $token): CompanyInvitation
    {
        $invitation = CompanyInvitation::with('company')
            ->whereToken($token)
            ->whereNull('accepted_at')
            ->where('expires_at', '>', now())
            ->first();

        abort_if($invitation === null, 404);

        return $invitation;
    }
}
```

- [ ] **Step 7: Add the routes**

Append inside the group in `routes/hr.php`:

```php
        Route::get('team', [TeamController::class, 'index'])->name('team.index');
        Route::post('team/invitations', [TeamController::class, 'store'])->name('team.invitations.store');
        Route::delete('team/invitations/{invitation}', [TeamController::class, 'destroy'])
            ->name('team.invitations.destroy');
```

and add the import:

```php
use App\Http\Controllers\Hr\TeamController;
```

Add the public invitation routes to `routes/web.php`, inside the existing `guest` group:

```php
    Route::get('invitations/{token}', [InvitationController::class, 'show'])->name('invitations.show');
    Route::post('invitations/{token}', [InvitationController::class, 'store'])->name('invitations.store');
```

with the import:

```php
use App\Http\Controllers\Auth\InvitationController;
```

- [ ] **Step 8: Build the team page**

`resources/js/pages/hr/team.tsx`:

```tsx
import { Form, Head, Link } from '@inertiajs/react';
import Heading from '@/components/heading';
import InputError from '@/components/input-error';
import { Badge } from '@/components/ui/badge';
import { Button } from '@/components/ui/button';
import { Input } from '@/components/ui/input';
import { Label } from '@/components/ui/label';
import { Spinner } from '@/components/ui/spinner';
import {
    Table,
    TableBody,
    TableCell,
    TableHead,
    TableHeader,
    TableRow,
} from '@/components/ui/table';
import { useClipboard } from '@/hooks/use-clipboard';
import { destroy, store } from '@/routes/hr/team/invitations';

type Member = {
    id: number;
    name: string;
    email: string;
    roles: string[];
};

type Invitation = {
    id: number;
    email: string;
    expires_at: string;
    expired: boolean;
    url: string;
};

type Props = {
    members: Member[];
    invitations: Invitation[];
};

export default function HrTeam({ members, invitations }: Props) {
    // useClipboard returns a tuple: [copiedText, copy]
    const [, copy] = useClipboard();

    return (
        <div className="flex flex-col gap-8 p-4">
            <Head title="Team" />
            <Heading title="Team" description="People who can manage postings and review candidates" />

            <Form {...store.form()} resetOnSuccess className="flex max-w-xl items-end gap-3">
                {({ processing, errors }) => (
                    <>
                        <div className="grid flex-1 gap-2">
                            <Label htmlFor="email">Invite by email</Label>
                            <Input id="email" name="email" type="email" required placeholder="colleague@company.com" />
                            <InputError message={errors.email} />
                        </div>
                        <Button type="submit" disabled={processing}>
                            {processing && <Spinner />}
                            Send invite
                        </Button>
                    </>
                )}
            </Form>

            <section className="flex flex-col gap-3">
                <h2 className="text-sm font-medium">Members</h2>
                <Table>
                    <TableHeader>
                        <TableRow>
                            <TableHead>Name</TableHead>
                            <TableHead>Email</TableHead>
                            <TableHead>Roles</TableHead>
                        </TableRow>
                    </TableHeader>
                    <TableBody>
                        {members.map((member) => (
                            <TableRow key={member.id}>
                                <TableCell className="font-medium">{member.name}</TableCell>
                                <TableCell>{member.email}</TableCell>
                                <TableCell className="space-x-1">
                                    {member.roles.map((role) => (
                                        <Badge key={role} variant="secondary">
                                            {role}
                                        </Badge>
                                    ))}
                                </TableCell>
                            </TableRow>
                        ))}
                    </TableBody>
                </Table>
            </section>

            <section className="flex flex-col gap-3">
                <h2 className="text-sm font-medium">Pending invitations</h2>
                {invitations.length === 0 ? (
                    <p className="text-sm text-muted-foreground">No pending invitations.</p>
                ) : (
                    <Table>
                        <TableHeader>
                            <TableRow>
                                <TableHead>Email</TableHead>
                                <TableHead>Expires</TableHead>
                                <TableHead className="text-right">Actions</TableHead>
                            </TableRow>
                        </TableHeader>
                        <TableBody>
                            {invitations.map((invitation) => (
                                <TableRow key={invitation.id}>
                                    <TableCell>{invitation.email}</TableCell>
                                    <TableCell>
                                        {invitation.expired ? (
                                            <Badge variant="destructive">Expired</Badge>
                                        ) : (
                                            invitation.expires_at
                                        )}
                                    </TableCell>
                                    <TableCell className="space-x-2 text-right">
                                        <Button variant="ghost" size="sm" onClick={() => copy(invitation.url)}>
                                            Copy link
                                        </Button>
                                        <Link
                                            href={destroy(invitation.id)}
                                            as="button"
                                            method="delete"
                                            className="text-sm text-destructive"
                                        >
                                            Revoke
                                        </Link>
                                    </TableCell>
                                </TableRow>
                            ))}
                        </TableBody>
                    </Table>
                )}
            </section>
        </div>
    );
}
```

`useClipboard` is a named export returning `[copiedText, copy]` — destructure it as a tuple, not an object.

- [ ] **Step 9: Build the invitation acceptance page**

`resources/js/pages/auth/accept-invitation.tsx`:

```tsx
import { Form, Head } from '@inertiajs/react';
import InputError from '@/components/input-error';
import PasswordInput from '@/components/password-input';
import { Button } from '@/components/ui/button';
import { Input } from '@/components/ui/input';
import { Label } from '@/components/ui/label';
import { Spinner } from '@/components/ui/spinner';
import { store } from '@/routes/invitations';

type Props = {
    token: string;
    email: string;
    companyName: string;
};

export default function AcceptInvitation({ token, email, companyName }: Props) {
    return (
        <>
            <Head title={`Join ${companyName}`} />

            <Form {...store.form(token)} className="flex flex-col gap-6">
                {({ processing, errors }) => (
                    <div className="grid gap-6">
                        <div className="grid gap-2">
                            <Label htmlFor="email">Email</Label>
                            <Input id="email" value={email} disabled readOnly />
                        </div>

                        <div className="grid gap-2">
                            <Label htmlFor="name">Your name</Label>
                            <Input id="name" name="name" required autoFocus autoComplete="name" />
                            <InputError message={errors.name} />
                        </div>

                        <div className="grid gap-2">
                            <Label htmlFor="password">Password</Label>
                            <PasswordInput id="password" name="password" required autoComplete="new-password" />
                            <InputError message={errors.password} />
                        </div>

                        <div className="grid gap-2">
                            <Label htmlFor="password_confirmation">Confirm password</Label>
                            <PasswordInput
                                id="password_confirmation"
                                name="password_confirmation"
                                required
                                autoComplete="new-password"
                            />
                            <InputError message={errors.password_confirmation} />
                        </div>

                        <Button type="submit" disabled={processing}>
                            {processing && <Spinner />}
                            Join {companyName}
                        </Button>
                    </div>
                )}
            </Form>
        </>
    );
}

AcceptInvitation.layout = {
    title: 'Accept your invitation',
    description: 'Set a password to join your team',
};
```

- [ ] **Step 10: Regenerate routes and run the test**

```bash
php artisan wayfinder:generate --with-form --no-interaction
php artisan test --compact --filter=TeamInvitationTest
```

Expected: PASS (7 tests).

- [ ] **Step 11: Format and commit**

```bash
vendor/bin/pint --dirty --format agent
git add -A
git commit -m "feat: add company team invitations"
```

---

## Task 5: Public job board and the apply flow

**Files:**
- Create: `app/Http/Controllers/JobBoardController.php`, `app/Http/Controllers/Candidate/ApplicationController.php`, `app/Http/Requests/Candidate/SubmitApplicationRequest.php`, `app/Actions/Applications/SubmitApplication.php`, `resources/js/layouts/public-layout.tsx`, `resources/js/pages/jobs/{index,show}.tsx`, `resources/js/pages/candidate/applications.tsx`
- Modify: `routes/web.php`, `resources/js/app.tsx`
- Test: `tests/Feature/Candidate/ApplyTest.php`

**Interfaces:**
- Consumes: `JobPosting::scopePublished()`, `Application`, `ApplicationStatus`, `ScreeningStatus` from Task 1
- Produces:
  - `SubmitApplication::handle(User $candidate, JobPosting $jobPosting, UploadedFile $cv): Application` — stores the CV, creates the application plus a `Pending` screening row, and dispatches `ScreenApplicationJob` (wired in Task 6; until then the dispatch line is absent)
  - Route names `jobs.index`, `jobs.show`, `jobs.apply`, `candidate.applications.index`

- [ ] **Step 1: Write the failing test**

Create `tests/Feature/Candidate/ApplyTest.php`:

```php
<?php

use App\Enums\ApplicationStatus;
use App\Enums\RoleName;
use App\Enums\ScreeningStatus;
use App\Models\Application;
use App\Models\JobPosting;
use App\Models\User;
use Illuminate\Http\UploadedFile;
use Illuminate\Support\Facades\Storage;

function candidateUser(): User
{
    $user = User::factory()->create();
    $user->assignRole(RoleName::Candidate->value);

    return $user;
}

test('the job board lists only published postings', function () {
    JobPosting::factory()->published()->create(['title' => 'Live Role']);
    JobPosting::factory()->create(['title' => 'Draft Role']);
    JobPosting::factory()->closed()->create(['title' => 'Closed Role']);

    $this->get(route('jobs.index'))
        ->assertOk()
        ->assertInertia(fn ($page) => $page->component('jobs/index')->has('jobPostings', 1));
});

test('a draft posting is not publicly viewable', function () {
    $posting = JobPosting::factory()->create();

    $this->get(route('jobs.show', $posting))->assertNotFound();
});

test('a candidate can apply with a pdf cv', function () {
    Storage::fake('local');

    $candidate = candidateUser();
    $posting = JobPosting::factory()->published()->create();

    $this->actingAs($candidate)
        ->post(route('jobs.apply', $posting), [
            'cv' => UploadedFile::fake()->create('resume.pdf', 200, 'application/pdf'),
        ])
        ->assertRedirect(route('candidate.applications.index'));

    $application = Application::firstOrFail();

    expect($application->user_id)->toBe($candidate->id)
        ->and($application->job_posting_id)->toBe($posting->id)
        ->and($application->status)->toBe(ApplicationStatus::Submitted)
        ->and($application->cv_original_name)->toBe('resume.pdf')
        ->and($application->latestScreening->status)->toBe(ScreeningStatus::Pending);

    Storage::disk('local')->assertExists($application->cv_path);
});

test('non pdf uploads are rejected', function () {
    Storage::fake('local');

    $candidate = candidateUser();
    $posting = JobPosting::factory()->published()->create();

    $this->actingAs($candidate)
        ->post(route('jobs.apply', $posting), [
            'cv' => UploadedFile::fake()->create('resume.docx', 200, 'application/msword'),
        ])
        ->assertSessionHasErrors('cv');

    expect(Application::count())->toBe(0);
});

test('a candidate cannot apply twice to the same posting', function () {
    Storage::fake('local');

    $candidate = candidateUser();
    $posting = JobPosting::factory()->published()->create();
    Application::factory()->create(['job_posting_id' => $posting->id, 'user_id' => $candidate->id]);

    $this->actingAs($candidate)
        ->post(route('jobs.apply', $posting), [
            'cv' => UploadedFile::fake()->create('resume.pdf', 200, 'application/pdf'),
        ])
        ->assertSessionHasErrors('cv');

    expect(Application::count())->toBe(1);
});

test('a candidate cannot apply to an unpublished posting', function () {
    Storage::fake('local');

    $candidate = candidateUser();
    $posting = JobPosting::factory()->create();

    $this->actingAs($candidate)
        ->post(route('jobs.apply', $posting), [
            'cv' => UploadedFile::fake()->create('resume.pdf', 200, 'application/pdf'),
        ])
        ->assertNotFound();
});

test('guests are sent to login when applying', function () {
    $posting = JobPosting::factory()->published()->create();

    $this->post(route('jobs.apply', $posting), [])->assertRedirect(route('login'));
});

test('a candidate sees only their own applications', function () {
    $candidate = candidateUser();
    Application::factory()->create(['user_id' => $candidate->id]);
    Application::factory()->create();

    $this->actingAs($candidate)
        ->get(route('candidate.applications.index'))
        ->assertOk()
        ->assertInertia(fn ($page) => $page->component('candidate/applications')->has('applications', 1));
});
```

- [ ] **Step 2: Run it to confirm it fails**

```bash
php artisan test --compact --filter=ApplyTest
```

Expected: FAIL — `Route [jobs.index] not defined`.

- [ ] **Step 3: Write the submit action**

```bash
php artisan make:class Actions/Applications/SubmitApplication --no-interaction
```

`app/Actions/Applications/SubmitApplication.php`:

```php
<?php

namespace App\Actions\Applications;

use App\Enums\ApplicationStatus;
use App\Enums\ScreeningStatus;
use App\Models\Application;
use App\Models\JobPosting;
use App\Models\User;
use Illuminate\Http\UploadedFile;
use Illuminate\Support\Facades\DB;
use Illuminate\Support\Str;

class SubmitApplication
{
    /**
     * Store the CV and record the application with a pending screening.
     */
    public function handle(User $candidate, JobPosting $jobPosting, UploadedFile $cv): Application
    {
        return DB::transaction(function () use ($candidate, $jobPosting, $cv): Application {
            $application = Application::create([
                'job_posting_id' => $jobPosting->id,
                'user_id' => $candidate->id,
                'cv_path' => 'pending',
                'cv_original_name' => $cv->getClientOriginalName(),
                'status' => ApplicationStatus::Submitted,
            ]);

            $path = $cv->storeAs(
                sprintf('cvs/%d/%d', $jobPosting->company_id, $application->id),
                Str::uuid()->toString().'.pdf',
                ['disk' => config('hr.cv.disk')],
            );

            $application->forceFill(['cv_path' => $path])->save();

            $application->screenings()->create([
                'status' => ScreeningStatus::Pending,
            ]);

            return $application;
        });
    }
}
```

The application row is created first so the storage path can include its id. The path is filled in immediately afterwards inside the same transaction.

- [ ] **Step 4: Write the form request**

```bash
php artisan make:request Candidate/SubmitApplicationRequest --no-interaction
```

`app/Http/Requests/Candidate/SubmitApplicationRequest.php`:

```php
<?php

namespace App\Http\Requests\Candidate;

use App\Models\Application;
use Illuminate\Contracts\Validation\ValidationRule;
use Illuminate\Foundation\Http\FormRequest;
use Illuminate\Validation\Validator;

class SubmitApplicationRequest extends FormRequest
{
    /**
     * @return array<string, ValidationRule|array<mixed>|string>
     */
    public function rules(): array
    {
        return [
            'cv' => ['required', 'file', 'mimes:pdf', 'max:'.config('hr.cv.max_kilobytes')],
        ];
    }

    /**
     * Reject a second application to the same posting.
     */
    public function after(): array
    {
        return [
            function (Validator $validator): void {
                $alreadyApplied = Application::query()
                    ->where('job_posting_id', $this->route('jobPosting')->id)
                    ->where('user_id', $this->user()->id)
                    ->exists();

                if ($alreadyApplied) {
                    $validator->errors()->add('cv', __('You have already applied to this role.'));
                }
            },
        ];
    }

    /**
     * @return array<string, string>
     */
    public function messages(): array
    {
        return [
            'cv.mimes' => __('Upload your CV as a PDF.'),
        ];
    }
}
```

- [ ] **Step 5: Write the public job board controller**

```bash
php artisan make:controller JobBoardController --no-interaction
```

`app/Http/Controllers/JobBoardController.php`:

```php
<?php

namespace App\Http\Controllers;

use App\Models\Application;
use App\Models\JobPosting;
use Illuminate\Http\Request;
use Inertia\Inertia;
use Inertia\Response;

class JobBoardController extends Controller
{
    /**
     * List every published posting across all companies.
     */
    public function index(): Response
    {
        $jobPostings = JobPosting::query()
            ->published()
            ->with('company:id,name')
            ->latest('published_at')
            ->get()
            ->map(fn (JobPosting $jobPosting): array => [
                'id' => $jobPosting->id,
                'title' => $jobPosting->title,
                'company' => $jobPosting->company->name,
                'location' => $jobPosting->location,
                'employment_type' => $jobPosting->employment_type->label(),
                'min_years_experience' => $jobPosting->min_years_experience,
            ]);

        return Inertia::render('jobs/index', [
            'jobPostings' => $jobPostings,
        ]);
    }

    /**
     * Show one published posting.
     */
    public function show(Request $request, JobPosting $jobPosting): Response
    {
        abort_unless($jobPosting->isOpen(), 404);

        $jobPosting->load('company:id,name');

        $hasApplied = $request->user() !== null && Application::query()
            ->where('job_posting_id', $jobPosting->id)
            ->where('user_id', $request->user()->id)
            ->exists();

        return Inertia::render('jobs/show', [
            'jobPosting' => [
                'id' => $jobPosting->id,
                'title' => $jobPosting->title,
                'company' => $jobPosting->company->name,
                'description' => $jobPosting->description,
                'requirements' => $jobPosting->requirements,
                'location' => $jobPosting->location,
                'employment_type' => $jobPosting->employment_type->label(),
                'min_years_experience' => $jobPosting->min_years_experience,
            ],
            'hasApplied' => $hasApplied,
        ]);
    }
}
```

- [ ] **Step 6: Write the candidate controller**

```bash
php artisan make:controller Candidate/ApplicationController --no-interaction
```

`app/Http/Controllers/Candidate/ApplicationController.php`:

```php
<?php

namespace App\Http\Controllers\Candidate;

use App\Actions\Applications\SubmitApplication;
use App\Http\Controllers\Controller;
use App\Http\Requests\Candidate\SubmitApplicationRequest;
use App\Models\Application;
use App\Models\JobPosting;
use Illuminate\Http\RedirectResponse;
use Illuminate\Http\Request;
use Inertia\Inertia;
use Inertia\Response;

class ApplicationController extends Controller
{
    /**
     * List the current candidate's applications.
     */
    public function index(Request $request): Response
    {
        $applications = Application::query()
            ->where('user_id', $request->user()->id)
            ->with('jobPosting.company:id,name')
            ->latest()
            ->get()
            ->map(fn (Application $application): array => [
                'id' => $application->id,
                'job_title' => $application->jobPosting->title,
                'company' => $application->jobPosting->company->name,
                'status' => $application->status->value,
                'status_label' => $application->status->label(),
                'rejection_reason' => $application->rejection_reason,
                'applied_at' => $application->created_at?->toDateString(),
            ]);

        return Inertia::render('candidate/applications', [
            'applications' => $applications,
        ]);
    }

    /**
     * Apply to a published posting.
     */
    public function store(
        SubmitApplicationRequest $request,
        JobPosting $jobPosting,
        SubmitApplication $submitApplication,
    ): RedirectResponse {
        abort_unless($jobPosting->isOpen(), 404);

        $submitApplication->handle($request->user(), $jobPosting, $request->file('cv'));

        Inertia::flash('toast', ['type' => 'success', 'message' => __('Application submitted. We will be in touch.')]);

        return to_route('candidate.applications.index');
    }
}
```

The `abort_unless` runs after validation because the Form Request resolves first. That is fine — an unpublished posting still 404s, and the test asserts exactly that with a valid PDF attached.

- [ ] **Step 7: Add the routes**

Add to `routes/web.php`, after the `guest` group:

```php
Route::get('jobs', [JobBoardController::class, 'index'])->name('jobs.index');
Route::get('jobs/{jobPosting}', [JobBoardController::class, 'show'])->name('jobs.show');

Route::middleware(['auth', 'verified'])->group(function () {
    Route::post('jobs/{jobPosting}/apply', [CandidateApplicationController::class, 'store'])
        ->middleware('throttle:10,1')
        ->name('jobs.apply');

    Route::get('my/applications', [CandidateApplicationController::class, 'index'])
        ->name('candidate.applications.index');
});
```

with the imports:

```php
use App\Http\Controllers\Candidate\ApplicationController as CandidateApplicationController;
use App\Http\Controllers\JobBoardController;
```

- [ ] **Step 8: Add a public layout and register it**

`resources/js/layouts/public-layout.tsx`:

```tsx
import { Link, usePage } from '@inertiajs/react';
import type { ReactNode } from 'react';
import AppLogoIcon from '@/components/app-logo-icon';
import { Button } from '@/components/ui/button';
import { dashboard, login, register } from '@/routes';
import { index as jobBoard } from '@/routes/jobs';

export default function PublicLayout({ children }: { children: ReactNode }) {
    const { auth } = usePage().props;

    return (
        <div className="flex min-h-screen flex-col">
            <header className="border-b">
                <div className="mx-auto flex w-full max-w-5xl items-center justify-between gap-4 px-4 py-4">
                    <Link href={jobBoard()} className="flex items-center gap-2 font-semibold">
                        <AppLogoIcon className="size-6" />
                        Open roles
                    </Link>

                    <nav className="flex items-center gap-2">
                        {auth?.user ? (
                            <Button asChild size="sm">
                                <Link href={dashboard()}>Dashboard</Link>
                            </Button>
                        ) : (
                            <>
                                <Button asChild variant="ghost" size="sm">
                                    <Link href={login()}>Log in</Link>
                                </Button>
                                <Button asChild size="sm">
                                    <Link href={register()}>Register</Link>
                                </Button>
                            </>
                        )}
                    </nav>
                </div>
            </header>

            <main className="mx-auto w-full max-w-5xl flex-1 px-4 py-8">{children}</main>
        </div>
    );
}
```

In `resources/js/app.tsx`, add a case to the layout resolver **above** the default, and import the layout:

```tsx
import PublicLayout from '@/layouts/public-layout';
```

```tsx
    layout: (name) => {
        switch (true) {
            case name === 'welcome':
                return null;
            case name.startsWith('auth/'):
                return AuthLayout;
            case name.startsWith('jobs/'):
                return PublicLayout;
            case name.startsWith('settings/'):
                return [AppLayout, SettingsLayout];
            default:
                return AppLayout;
        }
    },
```

Without this the public job pages render inside the authenticated sidebar shell and break for guests.

- [ ] **Step 9: Build the public job pages**

`resources/js/pages/jobs/index.tsx`:

```tsx
import { Head, Link } from '@inertiajs/react';
import { Badge } from '@/components/ui/badge';
import { Card, CardContent, CardHeader, CardTitle } from '@/components/ui/card';
import { show } from '@/routes/jobs';

type BoardPosting = {
    id: number;
    title: string;
    company: string;
    location: string;
    employment_type: string;
    min_years_experience: number;
};

type Props = {
    jobPostings: BoardPosting[];
};

export default function JobsIndex({ jobPostings }: Props) {
    return (
        <div className="flex flex-col gap-6">
            <Head title="Open roles" />
            <h1 className="text-2xl font-semibold">Open roles</h1>

            {jobPostings.length === 0 ? (
                <p className="text-sm text-muted-foreground">There are no open roles right now.</p>
            ) : (
                <div className="grid gap-4 sm:grid-cols-2">
                    {jobPostings.map((posting) => (
                        <Card key={posting.id}>
                            <CardHeader>
                                <CardTitle>
                                    <Link href={show(posting.id)} className="hover:underline">
                                        {posting.title}
                                    </Link>
                                </CardTitle>
                                <p className="text-sm text-muted-foreground">{posting.company}</p>
                            </CardHeader>
                            <CardContent className="flex flex-wrap gap-2">
                                <Badge variant="secondary">{posting.location}</Badge>
                                <Badge variant="secondary">{posting.employment_type}</Badge>
                                <Badge variant="secondary">{posting.min_years_experience}+ yrs</Badge>
                            </CardContent>
                        </Card>
                    ))}
                </div>
            )}
        </div>
    );
}
```

`resources/js/pages/jobs/show.tsx`:

```tsx
import { Form, Head, Link, usePage } from '@inertiajs/react';
import InputError from '@/components/input-error';
import { Badge } from '@/components/ui/badge';
import { Button } from '@/components/ui/button';
import { Input } from '@/components/ui/input';
import { Label } from '@/components/ui/label';
import { Spinner } from '@/components/ui/spinner';
import { login } from '@/routes';
import { apply } from '@/routes/jobs';

type Props = {
    jobPosting: {
        id: number;
        title: string;
        company: string;
        description: string;
        requirements: string;
        location: string;
        employment_type: string;
        min_years_experience: number;
    };
    hasApplied: boolean;
};

export default function JobsShow({ jobPosting, hasApplied }: Props) {
    const { auth } = usePage().props;

    return (
        <div className="flex flex-col gap-8">
            <Head title={jobPosting.title} />

            <header className="flex flex-col gap-2">
                <h1 className="text-2xl font-semibold">{jobPosting.title}</h1>
                <p className="text-muted-foreground">{jobPosting.company}</p>
                <div className="flex flex-wrap gap-2">
                    <Badge variant="secondary">{jobPosting.location}</Badge>
                    <Badge variant="secondary">{jobPosting.employment_type}</Badge>
                    <Badge variant="secondary">{jobPosting.min_years_experience}+ yrs experience</Badge>
                </div>
            </header>

            <section className="flex flex-col gap-2">
                <h2 className="font-medium">About the role</h2>
                <p className="whitespace-pre-line text-sm text-muted-foreground">{jobPosting.description}</p>
            </section>

            <section className="flex flex-col gap-2">
                <h2 className="font-medium">Requirements</h2>
                <p className="whitespace-pre-line text-sm text-muted-foreground">{jobPosting.requirements}</p>
            </section>

            <section className="flex flex-col gap-3 border-t pt-6">
                <h2 className="font-medium">Apply</h2>

                {!auth?.user && (
                    <p className="text-sm text-muted-foreground">
                        <Link href={login()} className="underline">
                            Log in
                        </Link>{' '}
                        to apply for this role.
                    </p>
                )}

                {auth?.user && hasApplied && (
                    <p className="text-sm text-muted-foreground">You have already applied to this role.</p>
                )}

                {auth?.user && !hasApplied && (
                    <Form {...apply.form(jobPosting.id)} className="flex max-w-md flex-col gap-3">
                        {({ processing, errors, progress }) => (
                            <>
                                <div className="grid gap-2">
                                    <Label htmlFor="cv">Your CV (PDF, max 5 MB)</Label>
                                    <Input id="cv" name="cv" type="file" accept="application/pdf" required />
                                    <InputError message={errors.cv} />
                                </div>

                                {progress && (
                                    <progress value={progress.percentage} max="100" className="w-full">
                                        {progress.percentage}%
                                    </progress>
                                )}

                                <Button type="submit" disabled={processing} className="w-fit">
                                    {processing && <Spinner />}
                                    Submit application
                                </Button>
                            </>
                        )}
                    </Form>
                )}
            </section>
        </div>
    );
}
```

- [ ] **Step 10: Build the candidate applications page**

`resources/js/pages/candidate/applications.tsx`:

```tsx
import { Head } from '@inertiajs/react';
import Heading from '@/components/heading';
import { Badge } from '@/components/ui/badge';
import { Card, CardContent, CardHeader, CardTitle } from '@/components/ui/card';

type CandidateApplication = {
    id: number;
    job_title: string;
    company: string;
    status: string;
    status_label: string;
    rejection_reason: string | null;
    applied_at: string | null;
};

type Props = {
    applications: CandidateApplication[];
};

const statusVariant = (status: string) => {
    if (status === 'shortlisted') {
        return 'default' as const;
    }

    if (status === 'rejected') {
        return 'destructive' as const;
    }

    return 'secondary' as const;
};

export default function CandidateApplications({ applications }: Props) {
    return (
        <div className="flex flex-col gap-6 p-4">
            <Head title="My applications" />
            <Heading title="My applications" description="Every role you have applied to" />

            {applications.length === 0 ? (
                <p className="text-sm text-muted-foreground">You have not applied to any roles yet.</p>
            ) : (
                <div className="flex flex-col gap-4">
                    {applications.map((application) => (
                        <Card key={application.id}>
                            <CardHeader className="flex flex-row items-start justify-between gap-4">
                                <div>
                                    <CardTitle>{application.job_title}</CardTitle>
                                    <p className="text-sm text-muted-foreground">
                                        {application.company} · applied {application.applied_at}
                                    </p>
                                </div>
                                <Badge variant={statusVariant(application.status)}>{application.status_label}</Badge>
                            </CardHeader>

                            {application.rejection_reason && (
                                <CardContent>
                                    <p className="text-sm font-medium">Feedback from the hiring team</p>
                                    <p className="mt-1 whitespace-pre-line text-sm text-muted-foreground">
                                        {application.rejection_reason}
                                    </p>
                                </CardContent>
                            )}
                        </Card>
                    ))}
                </div>
            )}
        </div>
    );
}
```

- [ ] **Step 11: Regenerate routes and run the tests**

```bash
php artisan wayfinder:generate --with-form --no-interaction
php artisan test --compact --filter="ApplyTest|DashboardTest"
```

Expected: PASS — 8 apply tests and 3 dashboard tests. The dashboard test was written in Task 2 and only now has both of its target routes.

- [ ] **Step 12: Format and commit**

```bash
vendor/bin/pint --dirty --format agent
git add -A
git commit -m "feat: add public job board and candidate apply flow"
```

---

## Task 6: AI screening pipeline

**Files:**
- Create: `app/Contracts/CvScreener.php`, `app/Data/ScreeningResult.php`, `app/Exceptions/{CvExtractionException,AiResponseException}.php`, `app/Services/Cv/CvTextExtractor.php`, `app/Services/Ai/OpenAiCvScreener.php`, `app/Jobs/ScreenApplicationJob.php`, `tests/Support/FakeCvScreener.php`
- Modify: `app/Providers/AppServiceProvider.php`, `app/Actions/Applications/SubmitApplication.php`
- Test: `tests/Feature/Screening/ScreenApplicationTest.php`, `tests/Unit/OpenAiCvScreenerTest.php`

**Interfaces:**
- Consumes: `Application`, `ApplicationScreening`, `JobPosting`, `ScreeningStatus`, `config('hr.ai.*')`
- Produces:
  - `ScreeningResult` readonly DTO with public properties `matchScore: int`, `summary: string`, `strengths: array<int,string>`, `gaps: array<int,string>`, `recommendation: string`, `model: string`, `inputTokens: ?int`, `outputTokens: ?int`
  - `CvScreener::screen(JobPosting $jobPosting, string $cvText): ScreeningResult`
  - `CvTextExtractor::extract(string $absolutePath): string`, throwing `CvExtractionException`
  - `ScreenApplicationJob::__construct(Application $application)`
  - `FakeCvScreener` for tests, constructed with an optional `ScreeningResult` or a `Throwable` to throw

- [ ] **Step 1: Write the test double**

Create `tests/Support/FakeCvScreener.php`:

```php
<?php

namespace Tests\Support;

use App\Contracts\CvScreener;
use App\Data\ScreeningResult;
use App\Models\JobPosting;
use Throwable;

class FakeCvScreener implements CvScreener
{
    public ?string $receivedCvText = null;

    public function __construct(
        private ?ScreeningResult $result = null,
        private ?Throwable $throwable = null,
    ) {}

    public function screen(JobPosting $jobPosting, string $cvText): ScreeningResult
    {
        $this->receivedCvText = $cvText;

        if ($this->throwable !== null) {
            throw $this->throwable;
        }

        return $this->result ?? new ScreeningResult(
            matchScore: 82,
            summary: 'Strong overlap with the required backend stack.',
            strengths: ['Six years of Laravel', 'Led a payments migration'],
            gaps: ['No Kubernetes experience'],
            recommendation: 'advance',
            model: 'fake-model',
            inputTokens: 1000,
            outputTokens: 200,
        );
    }
}
```

- [ ] **Step 2: Write the failing test**

Create `tests/Feature/Screening/ScreenApplicationTest.php`:

```php
<?php

use App\Contracts\CvScreener;
use App\Enums\RoleName;
use App\Enums\ScreeningStatus;
use App\Jobs\ScreenApplicationJob;
use App\Models\Application;
use App\Models\ApplicationScreening;
use App\Models\JobPosting;
use App\Models\User;
use Illuminate\Http\UploadedFile;
use Illuminate\Support\Facades\Queue;
use Illuminate\Support\Facades\Storage;
use Tests\Support\FakeCvScreener;

test('applying dispatches a screening job', function () {
    Storage::fake('local');
    Queue::fake();

    $candidate = User::factory()->create();
    $candidate->assignRole(RoleName::Candidate->value);
    $posting = JobPosting::factory()->published()->create();

    $this->actingAs($candidate)->post(route('jobs.apply', $posting), [
        'cv' => UploadedFile::fake()->create('resume.pdf', 200, 'application/pdf'),
    ]);

    Queue::assertPushed(ScreenApplicationJob::class);
});

test('the job records a completed screening', function () {
    $screener = new FakeCvScreener;
    $this->swap(CvScreener::class, $screener);

    $application = Application::factory()->create(['cv_text' => str_repeat('Laravel engineer. ', 40)]);
    ApplicationScreening::factory()->for($application)->create();

    (new ScreenApplicationJob($application))->handle(
        app(App\Services\Cv\CvTextExtractor::class),
        $screener,
    );

    $screening = $application->fresh()->latestScreening;

    expect($screening->status)->toBe(ScreeningStatus::Completed)
        ->and($screening->match_score)->toBe(82)
        ->and($screening->strengths)->toHaveCount(2)
        ->and($screening->gaps)->toHaveCount(1)
        ->and($screening->recommendation)->toBe('advance')
        ->and($screening->model)->toBe('fake-model')
        ->and($screening->completed_at)->not->toBeNull();
});

test('a cv with too little text fails without calling the provider', function () {
    $screener = new FakeCvScreener(throwable: new RuntimeException('should not be called'));
    $this->swap(CvScreener::class, $screener);

    $application = Application::factory()->create(['cv_text' => 'tiny']);
    ApplicationScreening::factory()->for($application)->create();

    (new ScreenApplicationJob($application))->handle(
        app(App\Services\Cv\CvTextExtractor::class),
        $screener,
    );

    $screening = $application->fresh()->latestScreening;

    expect($screening->status)->toBe(ScreeningStatus::Failed)
        ->and($screening->error)->toContain('Could not read text');
});

test('a provider failure marks the screening as failed', function () {
    $application = Application::factory()->create(['cv_text' => str_repeat('Laravel engineer. ', 40)]);
    ApplicationScreening::factory()->for($application)->create();

    (new ScreenApplicationJob($application))->failed(new RuntimeException('provider exploded'));

    $screening = $application->fresh()->latestScreening;

    expect($screening->status)->toBe(ScreeningStatus::Failed)
        ->and($screening->error)->toContain('provider exploded');
});

test('cv text is truncated to the configured maximum', function () {
    config()->set('hr.ai.max_cv_characters', 100);

    $screener = new FakeCvScreener;
    $this->swap(CvScreener::class, $screener);

    $application = Application::factory()->create(['cv_text' => str_repeat('a', 5000)]);
    ApplicationScreening::factory()->for($application)->create();

    (new ScreenApplicationJob($application))->handle(
        app(App\Services\Cv\CvTextExtractor::class),
        $screener,
    );

    expect(strlen($screener->receivedCvText))->toBeLessThanOrEqual(100);
});
```

- [ ] **Step 3: Run it to confirm it fails**

```bash
php artisan test --compact --filter=ScreenApplicationTest
```

Expected: FAIL — `Class "App\Contracts\CvScreener" not found`.

- [ ] **Step 4: Write the DTO, contract, and exceptions**

```bash
php artisan make:class Data/ScreeningResult --no-interaction
php artisan make:interface Contracts/CvScreener --no-interaction
php artisan make:exception CvExtractionException --no-interaction
php artisan make:exception AiResponseException --no-interaction
```

`app/Data/ScreeningResult.php`:

```php
<?php

namespace App\Data;

final readonly class ScreeningResult
{
    /**
     * @param  array<int, string>  $strengths
     * @param  array<int, string>  $gaps
     */
    public function __construct(
        public int $matchScore,
        public string $summary,
        public array $strengths,
        public array $gaps,
        public string $recommendation,
        public string $model,
        public ?int $inputTokens = null,
        public ?int $outputTokens = null,
    ) {}
}
```

`app/Contracts/CvScreener.php`:

```php
<?php

namespace App\Contracts;

use App\Data\ScreeningResult;
use App\Models\JobPosting;

interface CvScreener
{
    /**
     * Score a CV against a job posting.
     */
    public function screen(JobPosting $jobPosting, string $cvText): ScreeningResult;
}
```

`app/Exceptions/CvExtractionException.php`:

```php
<?php

namespace App\Exceptions;

use RuntimeException;

class CvExtractionException extends RuntimeException {}
```

`app/Exceptions/AiResponseException.php`:

```php
<?php

namespace App\Exceptions;

use RuntimeException;

class AiResponseException extends RuntimeException {}
```

- [ ] **Step 5: Write the PDF text extractor**

```bash
php artisan make:class Services/Cv/CvTextExtractor --no-interaction
```

`app/Services/Cv/CvTextExtractor.php`:

```php
<?php

namespace App\Services\Cv;

use App\Exceptions\CvExtractionException;
use Smalot\PdfParser\Parser;
use Throwable;

class CvTextExtractor
{
    /**
     * Pull readable text out of a PDF stored on disk.
     *
     * @throws CvExtractionException
     */
    public function extract(string $absolutePath): string
    {
        if (! is_readable($absolutePath)) {
            throw new CvExtractionException('The uploaded CV could not be read from storage.');
        }

        try {
            $text = (new Parser)->parseFile($absolutePath)->getText();
        } catch (Throwable $exception) {
            throw new CvExtractionException('This PDF could not be parsed.', previous: $exception);
        }

        return trim((string) preg_replace('/\s+/u', ' ', $text));
    }
}
```

- [ ] **Step 6: Write the OpenAI screener**

```bash
php artisan make:class Services/Ai/OpenAiCvScreener --no-interaction
```

`app/Services/Ai/OpenAiCvScreener.php`:

```php
<?php

namespace App\Services\Ai;

use App\Contracts\CvScreener;
use App\Data\ScreeningResult;
use App\Exceptions\AiResponseException;
use App\Models\JobPosting;
use JsonException;
use OpenAI\Laravel\Facades\OpenAI;

class OpenAiCvScreener implements CvScreener
{
    public function screen(JobPosting $jobPosting, string $cvText): ScreeningResult
    {
        $model = config('hr.ai.model');

        if (blank($model)) {
            throw new AiResponseException('OPENAI_MODEL is not set. Add it to your .env file.');
        }

        $response = OpenAI::responses()->create([
            'model' => $model,
            'input' => [
                ['role' => 'system', 'content' => $this->systemPrompt()],
                ['role' => 'user', 'content' => $this->userPrompt($jobPosting, $cvText)],
            ],
            'text' => [
                'format' => [
                    'type' => 'json_schema',
                    'name' => 'cv_screening',
                    'strict' => true,
                    'schema' => $this->schema(),
                ],
            ],
        ]);

        $payload = $this->decode($response->outputText);

        return new ScreeningResult(
            matchScore: (int) $payload['match_score'],
            summary: (string) $payload['summary'],
            strengths: array_values((array) $payload['strengths']),
            gaps: array_values((array) $payload['gaps']),
            recommendation: (string) $payload['recommendation'],
            model: $model,
            inputTokens: $response->usage->inputTokens ?? null,
            outputTokens: $response->usage->outputTokens ?? null,
        );
    }

    private function systemPrompt(): string
    {
        return <<<'PROMPT'
        You screen a single candidate CV against a single job posting for a hiring team.

        Judge only on evidence that appears in the CV. Never infer, assume, or invent
        experience that is not written there. If the CV does not mention a required
        skill, that is a gap, not a strength.

        match_score is a 0-100 estimate of how well this candidate fits this specific
        posting. Anchor it: 80+ means they clearly meet the requirements, 50-79 means a
        partial match with real gaps, below 50 means the core requirements are unmet.

        Write the summary in two or three sentences, addressed to the hiring team, in
        plain professional English. Keep each strength and gap to one short phrase.
        PROMPT;
    }

    private function userPrompt(JobPosting $jobPosting, string $cvText): string
    {
        return <<<PROMPT
        JOB TITLE: {$jobPosting->title}
        LOCATION: {$jobPosting->location}
        EMPLOYMENT TYPE: {$jobPosting->employment_type->label()}
        MINIMUM YEARS OF EXPERIENCE: {$jobPosting->min_years_experience}

        JOB DESCRIPTION:
        {$jobPosting->description}

        REQUIREMENTS:
        {$jobPosting->requirements}

        CANDIDATE CV:
        {$cvText}
        PROMPT;
    }

    /**
     * @return array<string, mixed>
     */
    private function schema(): array
    {
        return [
            'type' => 'object',
            'properties' => [
                'match_score' => ['type' => 'integer', 'minimum' => 0, 'maximum' => 100],
                'summary' => ['type' => 'string'],
                'strengths' => ['type' => 'array', 'items' => ['type' => 'string']],
                'gaps' => ['type' => 'array', 'items' => ['type' => 'string']],
                'recommendation' => ['type' => 'string', 'enum' => ['advance', 'hold', 'reject']],
            ],
            'required' => ['match_score', 'summary', 'strengths', 'gaps', 'recommendation'],
            'additionalProperties' => false,
        ];
    }

    /**
     * @return array<string, mixed>
     */
    private function decode(string $json): array
    {
        try {
            /** @var array<string, mixed> $payload */
            $payload = json_decode($json, true, flags: JSON_THROW_ON_ERROR);
        } catch (JsonException $exception) {
            throw new AiResponseException('The screening response was not valid JSON.', previous: $exception);
        }

        foreach (['match_score', 'summary', 'strengths', 'gaps', 'recommendation'] as $key) {
            if (! array_key_exists($key, $payload)) {
                throw new AiResponseException("The screening response is missing \"{$key}\".");
            }
        }

        return $payload;
    }
}
```

- [ ] **Step 7: Write the queue job**

```bash
php artisan make:job ScreenApplicationJob --no-interaction
```

`app/Jobs/ScreenApplicationJob.php`:

```php
<?php

namespace App\Jobs;

use App\Contracts\CvScreener;
use App\Enums\ScreeningStatus;
use App\Exceptions\CvExtractionException;
use App\Models\Application;
use App\Models\ApplicationScreening;
use App\Services\Cv\CvTextExtractor;
use Illuminate\Contracts\Queue\ShouldBeUnique;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Queue\Queueable;
use Illuminate\Support\Facades\Storage;
use Illuminate\Support\Str;
use Throwable;

class ScreenApplicationJob implements ShouldBeUnique, ShouldQueue
{
    use Queueable;

    public int $tries = 3;

    /**
     * Release the uniqueness lock even if the worker dies mid-job.
     */
    public int $uniqueFor = 600;

    public function __construct(public Application $application) {}

    public function uniqueId(): string
    {
        return (string) $this->application->id;
    }

    /**
     * @return array<int, int>
     */
    public function backoff(): array
    {
        return [10, 60];
    }

    public function handle(CvTextExtractor $extractor, CvScreener $screener): void
    {
        $application = $this->application->fresh(['jobPosting']);

        if ($application === null) {
            return;
        }

        $screening = $this->screeningFor($application);

        try {
            $text = filled($application->cv_text)
                ? $application->cv_text
                : $extractor->extract(Storage::disk(config('hr.cv.disk'))->path($application->cv_path));
        } catch (CvExtractionException $exception) {
            $this->markFailed($screening, $exception->getMessage());

            return;
        }

        if (Str::length($text) < config('hr.ai.min_cv_characters')) {
            $this->markFailed($screening, 'Could not read text from this PDF (it may be a scanned image).');

            return;
        }

        $text = Str::limit($text, config('hr.ai.max_cv_characters'), '');

        $application->forceFill(['cv_text' => $text])->save();

        $result = $screener->screen($application->jobPosting, $text);

        $screening->update([
            'status' => ScreeningStatus::Completed,
            'match_score' => $result->matchScore,
            'summary' => $result->summary,
            'strengths' => $result->strengths,
            'gaps' => $result->gaps,
            'recommendation' => $result->recommendation,
            'model' => $result->model,
            'input_tokens' => $result->inputTokens,
            'output_tokens' => $result->outputTokens,
            'error' => null,
            'completed_at' => now(),
        ]);
    }

    /**
     * Record the failure after the final retry.
     */
    public function failed(?Throwable $exception): void
    {
        $application = $this->application->fresh();

        if ($application === null) {
            return;
        }

        $this->markFailed(
            $this->screeningFor($application),
            $exception?->getMessage() ?? 'Screening failed.',
        );
    }

    private function screeningFor(Application $application): ApplicationScreening
    {
        return $application->screenings()->latest('id')->first()
            ?? $application->screenings()->create(['status' => ScreeningStatus::Pending]);
    }

    private function markFailed(ApplicationScreening $screening, string $error): void
    {
        $screening->update([
            'status' => ScreeningStatus::Failed,
            'error' => Str::limit($error, 500),
        ]);
    }
}
```

- [ ] **Step 8: Bind the contract**

In `app/Providers/AppServiceProvider.php`, add to `register()`:

```php
        $this->app->bind(\App\Contracts\CvScreener::class, \App\Services\Ai\OpenAiCvScreener::class);
```

- [ ] **Step 9: Dispatch the job when an application is submitted**

In `app/Actions/Applications/SubmitApplication.php`, add the import:

```php
use App\Jobs\ScreenApplicationJob;
```

and dispatch after the transaction commits. Replace the tail of `handle()` so the method reads:

```php
    public function handle(User $candidate, JobPosting $jobPosting, UploadedFile $cv): Application
    {
        $application = DB::transaction(function () use ($candidate, $jobPosting, $cv): Application {
            $application = Application::create([
                'job_posting_id' => $jobPosting->id,
                'user_id' => $candidate->id,
                'cv_path' => 'pending',
                'cv_original_name' => $cv->getClientOriginalName(),
                'status' => ApplicationStatus::Submitted,
            ]);

            $path = $cv->storeAs(
                sprintf('cvs/%d/%d', $jobPosting->company_id, $application->id),
                Str::uuid()->toString().'.pdf',
                ['disk' => config('hr.cv.disk')],
            );

            $application->forceFill(['cv_path' => $path])->save();

            $application->screenings()->create([
                'status' => ScreeningStatus::Pending,
            ]);

            return $application;
        });

        ScreenApplicationJob::dispatch($application)->afterCommit();

        return $application;
    }
```

- [ ] **Step 10: Write the OpenAI unit test**

Create `tests/Unit/OpenAiCvScreenerTest.php`:

```php
<?php

use App\Exceptions\AiResponseException;
use App\Models\JobPosting;
use App\Services\Ai\OpenAiCvScreener;

test('it refuses to run without a configured model', function () {
    config()->set('hr.ai.model', null);

    $jobPosting = new JobPosting([
        'title' => 'Backend Engineer',
        'description' => 'Build services.',
        'requirements' => 'PHP.',
        'employment_type' => 'full_time',
        'location' => 'Riyadh',
        'min_years_experience' => 3,
        'status' => 'published',
    ]);

    expect(fn () => (new OpenAiCvScreener)->screen($jobPosting, 'A CV.'))
        ->toThrow(AiResponseException::class, 'OPENAI_MODEL is not set.');
});
```

This test lives in `tests/Unit`, which does not get the `RefreshDatabase` trait, so it never touches the database — the unsaved `JobPosting` is only used to build the prompt, and the exception fires before any prompt is built.

- [ ] **Step 11: Run the tests**

```bash
php artisan test --compact --filter="ScreenApplicationTest|OpenAiCvScreenerTest"
```

Expected: PASS — 5 screening tests, 1 unit test.

- [ ] **Step 12: Smoke-test against the real API (optional but recommended)**

Start a worker and apply to a posting through the UI with a real PDF:

```bash
php artisan queue:work --once -v
```

Watch for a completed row:

```bash
php artisan tinker --execute 'App\Models\ApplicationScreening::latest()->first()->only(["status","match_score","summary","error"]);'
```

If it says `failed` with a provider error, check `OPENAI_MODEL` names a model your account can use.

- [ ] **Step 13: Format and commit**

```bash
vendor/bin/pint --dirty --format agent
git add -A
git commit -m "feat: score cvs against postings with queued openai screening"
```

---

## Task 7: HR candidate list and application detail

**Files:**
- Create: `app/Policies/ApplicationPolicy.php`, `app/Http/Controllers/Hr/{ApplicationController,CvDownloadController,RescreenController}.php`, `resources/js/components/{match-score-badge,screening-panel}.tsx`, `resources/js/pages/hr/jobs/applications.tsx`, `resources/js/pages/hr/applications/show.tsx`
- Modify: `routes/hr.php`, `resources/js/types/hr.ts`
- Test: `tests/Feature/Hr/ReviewApplicationsTest.php`

**Interfaces:**
- Consumes: `Application`, `ApplicationScreening`, `ScreenApplicationJob` from Tasks 1 and 6
- Produces:
  - `ApplicationPolicy` with `review`, `view`, `downloadCv`
  - Route names `hr.jobs.applications`, `hr.applications.show`, `hr.applications.cv`, `hr.applications.rescreen`
  - TS types `ApplicationRow`, `ScreeningDetail`, `ApplicationDetail` in `@/types/hr`
  - Wayfinder export `applications` from `@/routes/hr/jobs`, used by `hr/jobs/index.tsx` in Task 3

- [ ] **Step 1: Write the failing test**

Create `tests/Feature/Hr/ReviewApplicationsTest.php`:

```php
<?php

use App\Enums\RoleName;
use App\Jobs\ScreenApplicationJob;
use App\Models\Application;
use App\Models\ApplicationScreening;
use App\Models\Company;
use App\Models\JobPosting;
use App\Models\User;
use Illuminate\Support\Facades\Queue;
use Illuminate\Support\Facades\Storage;

function reviewer(?Company $company = null): User
{
    $company ??= Company::factory()->create();
    $user = User::factory()->create(['company_id' => $company->id]);
    $user->assignRole(RoleName::Hr->value);

    return $user;
}

test('hr sees candidates for their own posting ranked by score', function () {
    $user = reviewer();
    $posting = JobPosting::factory()->published()->create([
        'company_id' => $user->company_id,
        'created_by' => $user->id,
    ]);

    $low = Application::factory()->for($posting)->create();
    ApplicationScreening::factory()->for($low)->completed()->create(['match_score' => 40]);

    $high = Application::factory()->for($posting)->create();
    ApplicationScreening::factory()->for($high)->completed()->create(['match_score' => 91]);

    $this->actingAs($user)
        ->get(route('hr.jobs.applications', $posting))
        ->assertOk()
        ->assertInertia(fn ($page) => $page
            ->component('hr/jobs/applications')
            ->has('applications', 2)
            ->where('applications.0.id', $high->id)
            ->where('applications.0.screening.match_score', 91)
        );
});

test('hr cannot see candidates for another company posting', function () {
    $user = reviewer();
    $foreign = JobPosting::factory()->published()->create();

    $this->actingAs($user)
        ->get(route('hr.jobs.applications', $foreign))
        ->assertForbidden();
});

test('hr can open an application from their company', function () {
    $user = reviewer();
    $posting = JobPosting::factory()->create(['company_id' => $user->company_id, 'created_by' => $user->id]);
    $application = Application::factory()->for($posting)->create();
    ApplicationScreening::factory()->for($application)->completed()->create();

    $this->actingAs($user)
        ->get(route('hr.applications.show', $application))
        ->assertOk()
        ->assertInertia(fn ($page) => $page->component('hr/applications/show'));
});

test('hr cannot open an application from another company', function () {
    $user = reviewer();
    $application = Application::factory()->create();

    $this->actingAs($user)
        ->get(route('hr.applications.show', $application))
        ->assertForbidden();
});

test('a candidate cannot open the hr application page', function () {
    $candidate = User::factory()->create();
    $candidate->assignRole(RoleName::Candidate->value);
    $application = Application::factory()->create(['user_id' => $candidate->id]);

    $this->actingAs($candidate)
        ->get(route('hr.applications.show', $application))
        ->assertForbidden();
});

test('hr can download a cv from their own company', function () {
    Storage::fake('local');

    $user = reviewer();
    $posting = JobPosting::factory()->create(['company_id' => $user->company_id, 'created_by' => $user->id]);
    $application = Application::factory()->for($posting)->create(['cv_path' => 'cvs/1/1/file.pdf']);

    Storage::disk('local')->put('cvs/1/1/file.pdf', '%PDF-1.4 fake');

    $this->actingAs($user)
        ->get(route('hr.applications.cv', $application))
        ->assertOk()
        ->assertHeader('content-disposition', 'attachment; filename=resume.pdf');
});

test('hr cannot download a cv from another company', function () {
    $user = reviewer();
    $application = Application::factory()->create();

    $this->actingAs($user)
        ->get(route('hr.applications.cv', $application))
        ->assertForbidden();
});

test('hr can re-run a failed screening', function () {
    Queue::fake();

    $user = reviewer();
    $posting = JobPosting::factory()->create(['company_id' => $user->company_id, 'created_by' => $user->id]);
    $application = Application::factory()->for($posting)->create();
    ApplicationScreening::factory()->for($application)->failed()->create();

    $this->actingAs($user)
        ->post(route('hr.applications.rescreen', $application))
        ->assertRedirect();

    Queue::assertPushed(ScreenApplicationJob::class);

    expect($application->fresh()->latestScreening->status->value)->toBe('pending');
});
```

- [ ] **Step 2: Run it to confirm it fails**

```bash
php artisan test --compact --filter=ReviewApplicationsTest
```

Expected: FAIL — `Route [hr.jobs.applications] not defined`.

- [ ] **Step 3: Write the policy**

```bash
php artisan make:policy ApplicationPolicy --model=Application --no-interaction
```

Replace `app/Policies/ApplicationPolicy.php`:

```php
<?php

namespace App\Policies;

use App\Enums\PermissionName;
use App\Models\Application;
use App\Models\User;

class ApplicationPolicy
{
    /**
     * HR of the owning company may review this application.
     */
    public function review(User $user, Application $application): bool
    {
        return $user->can(PermissionName::ReviewApplications->value)
            && $user->company_id !== null
            && $user->company_id === $application->jobPosting->company_id;
    }

    /**
     * Reviewers, plus the candidate who submitted it.
     */
    public function view(User $user, Application $application): bool
    {
        return $this->review($user, $application) || $user->id === $application->user_id;
    }

    /**
     * Only reviewers may pull the CV file down.
     */
    public function downloadCv(User $user, Application $application): bool
    {
        return $this->review($user, $application);
    }
}
```

- [ ] **Step 4: Write the review controllers**

```bash
php artisan make:controller Hr/ApplicationController --no-interaction
php artisan make:controller Hr/CvDownloadController --invokable --no-interaction
php artisan make:controller Hr/RescreenController --invokable --no-interaction
```

`app/Http/Controllers/Hr/ApplicationController.php`:

```php
<?php

namespace App\Http\Controllers\Hr;

use App\Http\Controllers\Controller;
use App\Models\Application;
use App\Models\ApplicationScreening;
use App\Models\JobPosting;
use Illuminate\Support\Facades\Gate;
use Inertia\Inertia;
use Inertia\Response;

class ApplicationController extends Controller
{
    /**
     * List every candidate for one posting, best match first.
     */
    public function index(JobPosting $jobPosting): Response
    {
        Gate::authorize('view', $jobPosting);

        $applications = $jobPosting->applications()
            ->with(['candidate:id,name,email', 'latestScreening'])
            ->get()
            ->sortByDesc(fn (Application $application): int => $application->latestScreening?->match_score ?? -1)
            ->values()
            ->map(fn (Application $application): array => [
                'id' => $application->id,
                'candidate_name' => $application->candidate->name,
                'candidate_email' => $application->candidate->email,
                'status' => $application->status->value,
                'status_label' => $application->status->label(),
                'applied_at' => $application->created_at?->toDateString(),
                'screening' => $this->screeningPayload($application->latestScreening),
            ]);

        return Inertia::render('hr/jobs/applications', [
            'jobPosting' => [
                'id' => $jobPosting->id,
                'title' => $jobPosting->title,
            ],
            'applications' => $applications,
            'hasPendingScreenings' => $jobPosting->applications()
                ->whereHas('screenings', fn ($query) => $query->where('status', 'pending'))
                ->exists(),
        ]);
    }

    /**
     * Show one candidate with their screening and decision controls.
     */
    public function show(Application $application): Response
    {
        Gate::authorize('review', $application);

        $application->load(['candidate:id,name,email', 'jobPosting:id,title', 'latestScreening', 'decider:id,name']);

        return Inertia::render('hr/applications/show', [
            'application' => [
                'id' => $application->id,
                'candidate_name' => $application->candidate->name,
                'candidate_email' => $application->candidate->email,
                'cv_original_name' => $application->cv_original_name,
                'status' => $application->status->value,
                'status_label' => $application->status->label(),
                'rejection_reason' => $application->rejection_reason,
                'applied_at' => $application->created_at?->toDateString(),
                'decided_by' => $application->decider?->name,
                'job_posting' => [
                    'id' => $application->jobPosting->id,
                    'title' => $application->jobPosting->title,
                ],
                'screening' => $this->screeningPayload($application->latestScreening),
            ],
        ]);
    }

    /**
     * @return array{status: string, status_label: string, match_score: int|null, summary: string|null, strengths: array<int, string>, gaps: array<int, string>, recommendation: string|null, error: string|null}|null
     */
    private function screeningPayload(?ApplicationScreening $screening): ?array
    {
        if ($screening === null) {
            return null;
        }

        return [
            'status' => $screening->status->value,
            'status_label' => $screening->status->label(),
            'match_score' => $screening->match_score,
            'summary' => $screening->summary,
            'strengths' => $screening->strengths ?? [],
            'gaps' => $screening->gaps ?? [],
            'recommendation' => $screening->recommendation,
            'error' => $screening->error,
        ];
    }
}
```

`app/Http/Controllers/Hr/CvDownloadController.php`:

```php
<?php

namespace App\Http\Controllers\Hr;

use App\Http\Controllers\Controller;
use App\Models\Application;
use Illuminate\Support\Facades\Gate;
use Illuminate\Support\Facades\Storage;
use Symfony\Component\HttpFoundation\StreamedResponse;

class CvDownloadController extends Controller
{
    /**
     * Stream a CV from the private disk. The path is never exposed.
     */
    public function __invoke(Application $application): StreamedResponse
    {
        Gate::authorize('downloadCv', $application);

        $disk = Storage::disk(config('hr.cv.disk'));

        abort_unless($disk->exists($application->cv_path), 404);

        return $disk->download($application->cv_path, $application->cv_original_name);
    }
}
```

`app/Http/Controllers/Hr/RescreenController.php`:

```php
<?php

namespace App\Http\Controllers\Hr;

use App\Enums\ScreeningStatus;
use App\Http\Controllers\Controller;
use App\Jobs\ScreenApplicationJob;
use App\Models\Application;
use Illuminate\Http\RedirectResponse;
use Illuminate\Support\Facades\Gate;
use Inertia\Inertia;

class RescreenController extends Controller
{
    /**
     * Queue a fresh screening for an application.
     */
    public function __invoke(Application $application): RedirectResponse
    {
        Gate::authorize('review', $application);

        $application->screenings()->create(['status' => ScreeningStatus::Pending]);

        ScreenApplicationJob::dispatch($application);

        Inertia::flash('toast', ['type' => 'info', 'message' => __('Screening queued.')]);

        return back();
    }
}
```

- [ ] **Step 5: Add the routes**

Add inside the group in `routes/hr.php`:

```php
        Route::get('jobs/{jobPosting}/applications', [ApplicationController::class, 'index'])
            ->name('jobs.applications');

        Route::get('applications/{application}', [ApplicationController::class, 'show'])
            ->name('applications.show');

        Route::get('applications/{application}/cv', CvDownloadController::class)
            ->name('applications.cv');

        Route::post('applications/{application}/rescreen', RescreenController::class)
            ->name('applications.rescreen');
```

with the imports:

```php
use App\Http\Controllers\Hr\ApplicationController;
use App\Http\Controllers\Hr\CvDownloadController;
use App\Http\Controllers\Hr\RescreenController;
```

> Register `jobs/{jobPosting}/applications` **after** `Route::resource('jobs', ...)`. The resource's `jobs/{jobPosting}/edit` route is distinct, so there is no conflict, but keeping the order stable avoids surprises if you add a `show` route later.

- [ ] **Step 6: Add the TypeScript types**

Append to `resources/js/types/hr.ts`:

```typescript
export type ScreeningDetail = {
    status: string;
    status_label: string;
    match_score: number | null;
    summary: string | null;
    strengths: string[];
    gaps: string[];
    recommendation: string | null;
    error: string | null;
};

export type ApplicationRow = {
    id: number;
    candidate_name: string;
    candidate_email: string;
    status: string;
    status_label: string;
    applied_at: string | null;
    screening: ScreeningDetail | null;
};

export type ApplicationDetail = {
    id: number;
    candidate_name: string;
    candidate_email: string;
    cv_original_name: string;
    status: string;
    status_label: string;
    rejection_reason: string | null;
    applied_at: string | null;
    decided_by: string | null;
    job_posting: { id: number; title: string };
    screening: ScreeningDetail | null;
};
```

- [ ] **Step 7: Build the score badge and screening panel**

`resources/js/components/match-score-badge.tsx`:

```tsx
import { cn } from '@/lib/utils';

type Props = {
    score: number;
    className?: string;
};

/**
 * Fixed bands so a score means the same thing on every screen:
 * 75+ strong, 50-74 partial, below 50 weak.
 */
const bandClass = (score: number): string => {
    if (score >= 75) {
        return 'bg-emerald-100 text-emerald-900 dark:bg-emerald-950 dark:text-emerald-200';
    }

    if (score >= 50) {
        return 'bg-amber-100 text-amber-900 dark:bg-amber-950 dark:text-amber-200';
    }

    return 'bg-rose-100 text-rose-900 dark:bg-rose-950 dark:text-rose-200';
};

export default function MatchScoreBadge({ score, className }: Props) {
    return (
        <span
            className={cn(
                'inline-flex min-w-12 items-center justify-center rounded-md px-2 py-0.5 text-sm font-semibold tabular-nums',
                bandClass(score),
                className,
            )}
        >
            {score}
        </span>
    );
}
```

`resources/js/components/screening-panel.tsx`:

```tsx
import { Link } from '@inertiajs/react';
import MatchScoreBadge from '@/components/match-score-badge';
import { Card, CardContent, CardHeader, CardTitle } from '@/components/ui/card';
import { Skeleton } from '@/components/ui/skeleton';
import { rescreen } from '@/routes/hr/applications';
import type { ScreeningDetail } from '@/types/hr';

type Props = {
    applicationId: number;
    screening: ScreeningDetail | null;
};

export default function ScreeningPanel({ applicationId, screening }: Props) {
    if (!screening || screening.status === 'pending') {
        return (
            <Card>
                <CardHeader>
                    <CardTitle>AI screening</CardTitle>
                </CardHeader>
                <CardContent className="flex flex-col gap-2">
                    <Skeleton className="h-4 w-3/4" />
                    <Skeleton className="h-4 w-1/2" />
                    <p className="text-sm text-muted-foreground">Screening this CV against the posting…</p>
                </CardContent>
            </Card>
        );
    }

    if (screening.status === 'failed') {
        return (
            <Card>
                <CardHeader>
                    <CardTitle>AI screening</CardTitle>
                </CardHeader>
                <CardContent className="flex flex-col items-start gap-3">
                    <p className="text-sm text-muted-foreground">{screening.error ?? 'Screening failed.'}</p>
                    <Link
                        href={rescreen(applicationId)}
                        method="post"
                        as="button"
                        className="text-sm underline"
                    >
                        Re-run screening
                    </Link>
                </CardContent>
            </Card>
        );
    }

    return (
        <Card>
            <CardHeader className="flex flex-row items-center justify-between">
                <CardTitle>AI screening</CardTitle>
                {screening.match_score !== null && <MatchScoreBadge score={screening.match_score} />}
            </CardHeader>
            <CardContent className="flex flex-col gap-4">
                <p className="text-sm">{screening.summary}</p>

                <div className="grid gap-4 sm:grid-cols-2">
                    <div>
                        <h3 className="text-sm font-medium">Strengths</h3>
                        <ul className="mt-1 list-inside list-disc text-sm text-muted-foreground">
                            {screening.strengths.map((item) => (
                                <li key={item}>{item}</li>
                            ))}
                        </ul>
                    </div>
                    <div>
                        <h3 className="text-sm font-medium">Gaps</h3>
                        <ul className="mt-1 list-inside list-disc text-sm text-muted-foreground">
                            {screening.gaps.map((item) => (
                                <li key={item}>{item}</li>
                            ))}
                        </ul>
                    </div>
                </div>

                {screening.recommendation && (
                    <p className="text-sm text-muted-foreground">
                        Recommendation: <span className="font-medium">{screening.recommendation}</span>
                    </p>
                )}
            </CardContent>
        </Card>
    );
}
```

- [ ] **Step 8: Build the candidate list page**

`resources/js/pages/hr/jobs/applications.tsx`:

```tsx
import { Head, Link, usePoll } from '@inertiajs/react';
import Heading from '@/components/heading';
import MatchScoreBadge from '@/components/match-score-badge';
import { Badge } from '@/components/ui/badge';
import { Button } from '@/components/ui/button';
import { Skeleton } from '@/components/ui/skeleton';
import {
    Table,
    TableBody,
    TableCell,
    TableHead,
    TableHeader,
    TableRow,
} from '@/components/ui/table';
import { show } from '@/routes/hr/applications';
import type { ApplicationRow } from '@/types/hr';

type Props = {
    jobPosting: { id: number; title: string };
    applications: ApplicationRow[];
    hasPendingScreenings: boolean;
};

export default function HrJobApplications({ jobPosting, applications, hasPendingScreenings }: Props) {
    // Only poll while at least one CV is still being screened; it stops itself
    // once the server reports no pending screenings.
    usePoll(5000, { only: ['applications', 'hasPendingScreenings'] }, { autoStart: hasPendingScreenings });

    return (
        <div className="flex flex-col gap-6 p-4">
            <Head title={`Candidates · ${jobPosting.title}`} />
            <Heading title="Candidates" description={jobPosting.title} />

            {applications.length === 0 ? (
                <p className="text-sm text-muted-foreground">No applications yet.</p>
            ) : (
                <Table>
                    <TableHeader>
                        <TableRow>
                            <TableHead className="w-20">Match</TableHead>
                            <TableHead>Candidate</TableHead>
                            <TableHead>Summary</TableHead>
                            <TableHead>Status</TableHead>
                            <TableHead className="text-right">Applied</TableHead>
                            <TableHead />
                        </TableRow>
                    </TableHeader>
                    <TableBody>
                        {applications.map((application) => (
                            <TableRow key={application.id}>
                                <TableCell>
                                    {application.screening?.status === 'completed' &&
                                    application.screening.match_score !== null ? (
                                        <MatchScoreBadge score={application.screening.match_score} />
                                    ) : application.screening?.status === 'failed' ? (
                                        <Badge variant="destructive">Failed</Badge>
                                    ) : (
                                        <Skeleton className="h-6 w-12" />
                                    )}
                                </TableCell>
                                <TableCell>
                                    <div className="font-medium">{application.candidate_name}</div>
                                    <div className="text-sm text-muted-foreground">
                                        {application.candidate_email}
                                    </div>
                                </TableCell>
                                <TableCell className="max-w-md text-sm text-muted-foreground">
                                    {application.screening?.summary ?? '—'}
                                </TableCell>
                                <TableCell>
                                    <Badge
                                        variant={
                                            application.status === 'shortlisted'
                                                ? 'default'
                                                : application.status === 'rejected'
                                                  ? 'destructive'
                                                  : 'secondary'
                                        }
                                    >
                                        {application.status_label}
                                    </Badge>
                                </TableCell>
                                <TableCell className="text-right text-sm">{application.applied_at}</TableCell>
                                <TableCell className="text-right">
                                    <Button variant="ghost" size="sm" asChild>
                                        <Link href={show(application.id)}>Review</Link>
                                    </Button>
                                </TableCell>
                            </TableRow>
                        ))}
                    </TableBody>
                </Table>
            )}
        </div>
    );
}
```

- [ ] **Step 9: Build the application detail page**

This version has no decision buttons yet — Task 8 adds them.

`resources/js/pages/hr/applications/show.tsx`:

```tsx
import { Head, Link } from '@inertiajs/react';
import Heading from '@/components/heading';
import ScreeningPanel from '@/components/screening-panel';
import { Badge } from '@/components/ui/badge';
import { Button } from '@/components/ui/button';
import { Card, CardContent, CardHeader, CardTitle } from '@/components/ui/card';
import { cv } from '@/routes/hr/applications';
import { applications as jobApplications } from '@/routes/hr/jobs';
import type { ApplicationDetail } from '@/types/hr';

type Props = {
    application: ApplicationDetail;
};

export default function HrApplicationShow({ application }: Props) {
    return (
        <div className="flex flex-col gap-6 p-4">
            <Head title={application.candidate_name} />

            <div className="flex flex-wrap items-start justify-between gap-4">
                <Heading
                    title={application.candidate_name}
                    description={`${application.candidate_email} · applied ${application.applied_at}`}
                />
                <div className="flex items-center gap-2">
                    <Badge
                        variant={
                            application.status === 'shortlisted'
                                ? 'default'
                                : application.status === 'rejected'
                                  ? 'destructive'
                                  : 'secondary'
                        }
                    >
                        {application.status_label}
                    </Badge>
                    <Button variant="outline" size="sm" asChild>
                        <a href={cv.url(application.id)}>Download CV</a>
                    </Button>
                    <Button variant="ghost" size="sm" asChild>
                        <Link href={jobApplications(application.job_posting.id)}>Back to candidates</Link>
                    </Button>
                </div>
            </div>

            <ScreeningPanel applicationId={application.id} screening={application.screening} />

            {application.rejection_reason && (
                <Card>
                    <CardHeader>
                        <CardTitle>Rejection message sent</CardTitle>
                    </CardHeader>
                    <CardContent>
                        <p className="whitespace-pre-line text-sm text-muted-foreground">
                            {application.rejection_reason}
                        </p>
                    </CardContent>
                </Card>
            )}
        </div>
    );
}
```

The CV link is a plain `<a>`, not `<Link>` — a file download must not go through Inertia's XHR client.

- [ ] **Step 10: Restore the Candidates button in the jobs index**

If you removed it in Task 3, re-add the import and the button now that the route exists:

```tsx
import { applications } from '@/routes/hr/jobs';
```

```tsx
                                    <Button variant="ghost" size="sm" asChild>
                                        <Link href={applications(posting.id)}>Candidates</Link>
                                    </Button>
```

- [ ] **Step 11: Regenerate routes and run the tests**

```bash
php artisan wayfinder:generate --with-form --no-interaction
php artisan test --compact --filter=ReviewApplicationsTest
```

Expected: PASS (8 tests).

- [ ] **Step 12: Format and commit**

```bash
vendor/bin/pint --dirty --format agent
git add -A
git commit -m "feat: add hr candidate review screens with match scores"
```

---

## Task 8: Shortlist, reject, and the AI rejection message

**Files:**
- Create: `app/Contracts/RejectionWriter.php`, `app/Services/Ai/OpenAiRejectionWriter.php`, `app/Actions/Applications/{ShortlistApplication,RejectApplication}.php`, `app/Http/Requests/Hr/{RejectApplicationRequest,EnhanceRejectionRequest}.php`, `app/Http/Controllers/Hr/{ApplicationDecisionController,RejectionMessageController}.php`, `tests/Support/FakeRejectionWriter.php`, `resources/js/components/reject-dialog.tsx`
- Modify: `routes/hr.php`, `app/Providers/AppServiceProvider.php`, `resources/js/pages/hr/applications/show.tsx`
- Test: `tests/Feature/Hr/DecideApplicationTest.php`

**Interfaces:**
- Consumes: `Application`, `ApplicationStatus`, `ApplicationPolicy::review` from Tasks 1 and 7
- Produces:
  - `RejectionWriter::draft(Application $application): string` and `RejectionWriter::enhance(Application $application, string $draft): string`
  - `ShortlistApplication::handle(User $decider, Application $application): Application`
  - `RejectApplication::handle(User $decider, Application $application, string $reason): Application`
  - Route names `hr.applications.shortlist`, `hr.applications.reject`, `hr.applications.rejection.draft`, `hr.applications.rejection.enhance`
  - The two rejection endpoints return JSON `{"message": string}`

- [ ] **Step 1: Write the test double**

Create `tests/Support/FakeRejectionWriter.php`:

```php
<?php

namespace Tests\Support;

use App\Contracts\RejectionWriter;
use App\Models\Application;

class FakeRejectionWriter implements RejectionWriter
{
    public ?string $receivedDraft = null;

    public function draft(Application $application): string
    {
        return "Dear {$application->candidate->name}, thank you for applying. After review we are moving forward with other candidates.";
    }

    public function enhance(Application $application, string $draft): string
    {
        $this->receivedDraft = $draft;

        return "Polished: {$draft}";
    }
}
```

- [ ] **Step 2: Write the failing test**

Create `tests/Feature/Hr/DecideApplicationTest.php`:

```php
<?php

use App\Contracts\RejectionWriter;
use App\Enums\ApplicationStatus;
use App\Enums\RoleName;
use App\Models\Application;
use App\Models\ApplicationScreening;
use App\Models\Company;
use App\Models\JobPosting;
use App\Models\User;
use Tests\Support\FakeRejectionWriter;

function decider(?Company $company = null): User
{
    $company ??= Company::factory()->create();
    $user = User::factory()->create(['company_id' => $company->id]);
    $user->assignRole(RoleName::Hr->value);

    return $user;
}

function applicationFor(User $user): Application
{
    $posting = JobPosting::factory()->create([
        'company_id' => $user->company_id,
        'created_by' => $user->id,
    ]);

    $application = Application::factory()->for($posting)->create();
    ApplicationScreening::factory()->for($application)->completed()->create();

    return $application;
}

test('hr can shortlist a candidate', function () {
    $user = decider();
    $application = applicationFor($user);

    $this->actingAs($user)
        ->post(route('hr.applications.shortlist', $application))
        ->assertRedirect();

    $application->refresh();

    expect($application->status)->toBe(ApplicationStatus::Shortlisted)
        ->and($application->decided_by)->toBe($user->id)
        ->and($application->decided_at)->not->toBeNull();
});

test('hr can reject a candidate with a message', function () {
    $user = decider();
    $application = applicationFor($user);

    $this->actingAs($user)
        ->post(route('hr.applications.reject', $application), [
            'rejection_reason' => 'Thank you for your time. We are moving ahead with candidates who have more billing experience.',
        ])
        ->assertRedirect();

    $application->refresh();

    expect($application->status)->toBe(ApplicationStatus::Rejected)
        ->and($application->rejection_reason)->toContain('billing experience')
        ->and($application->decided_by)->toBe($user->id);
});

test('rejecting without a message fails validation', function () {
    $user = decider();
    $application = applicationFor($user);

    $this->actingAs($user)
        ->post(route('hr.applications.reject', $application), ['rejection_reason' => 'too short'])
        ->assertSessionHasErrors('rejection_reason');

    expect($application->fresh()->status)->toBe(ApplicationStatus::Submitted);
});

test('hr cannot decide on another company application', function () {
    $user = decider();
    $foreign = Application::factory()->create();

    $this->actingAs($user)
        ->post(route('hr.applications.shortlist', $foreign))
        ->assertForbidden();

    $this->actingAs($user)
        ->post(route('hr.applications.reject', $foreign), [
            'rejection_reason' => 'A perfectly valid rejection message of sufficient length.',
        ])
        ->assertForbidden();
});

test('the draft endpoint returns an ai written message', function () {
    $this->swap(RejectionWriter::class, new FakeRejectionWriter);

    $user = decider();
    $application = applicationFor($user);

    $this->actingAs($user)
        ->postJson(route('hr.applications.rejection.draft', $application))
        ->assertOk()
        ->assertJsonStructure(['message'])
        ->assertJsonFragment(['message' => "Dear {$application->candidate->name}, thank you for applying. After review we are moving forward with other candidates."]);
});

test('the enhance endpoint rewrites the hr message', function () {
    $writer = new FakeRejectionWriter;
    $this->swap(RejectionWriter::class, $writer);

    $user = decider();
    $application = applicationFor($user);

    $this->actingAs($user)
        ->postJson(route('hr.applications.rejection.enhance', $application), [
            'message' => 'not enough php experience',
        ])
        ->assertOk()
        ->assertJsonFragment(['message' => 'Polished: not enough php experience']);

    expect($writer->receivedDraft)->toBe('not enough php experience');
});

test('the enhance endpoint requires a message', function () {
    $this->swap(RejectionWriter::class, new FakeRejectionWriter);

    $user = decider();
    $application = applicationFor($user);

    $this->actingAs($user)
        ->postJson(route('hr.applications.rejection.enhance', $application), ['message' => ''])
        ->assertStatus(422);
});

test('hr cannot draft a rejection for another company', function () {
    $this->swap(RejectionWriter::class, new FakeRejectionWriter);

    $user = decider();
    $foreign = Application::factory()->create();

    $this->actingAs($user)
        ->postJson(route('hr.applications.rejection.draft', $foreign))
        ->assertForbidden();
});

test('the candidate sees the rejection reason', function () {
    $user = decider();
    $application = applicationFor($user);

    $this->actingAs($user)->post(route('hr.applications.reject', $application), [
        'rejection_reason' => 'We were looking for deeper experience with high-volume queues.',
    ]);

    $candidate = $application->candidate;
    $candidate->assignRole(RoleName::Candidate->value);

    $this->actingAs($candidate)
        ->get(route('candidate.applications.index'))
        ->assertOk()
        ->assertInertia(fn ($page) => $page
            ->where('applications.0.rejection_reason', 'We were looking for deeper experience with high-volume queues.')
        );
});
```

- [ ] **Step 3: Run it to confirm it fails**

```bash
php artisan test --compact --filter=DecideApplicationTest
```

Expected: FAIL — `Route [hr.applications.shortlist] not defined`.

- [ ] **Step 4: Write the contract**

```bash
php artisan make:interface Contracts/RejectionWriter --no-interaction
```

`app/Contracts/RejectionWriter.php`:

```php
<?php

namespace App\Contracts;

use App\Models\Application;

interface RejectionWriter
{
    /**
     * Write a rejection message from scratch for this application.
     */
    public function draft(Application $application): string;

    /**
     * Rewrite the hiring team's own wording without adding new reasons.
     */
    public function enhance(Application $application, string $draft): string;
}
```

- [ ] **Step 5: Write the OpenAI implementation**

```bash
php artisan make:class Services/Ai/OpenAiRejectionWriter --no-interaction
```

`app/Services/Ai/OpenAiRejectionWriter.php`:

```php
<?php

namespace App\Services\Ai;

use App\Contracts\RejectionWriter;
use App\Exceptions\AiResponseException;
use App\Models\Application;
use OpenAI\Laravel\Facades\OpenAI;

class OpenAiRejectionWriter implements RejectionWriter
{
    public function draft(Application $application): string
    {
        $application->loadMissing(['candidate:id,name', 'jobPosting.company:id,name', 'latestScreening']);

        $gaps = $application->latestScreening?->gaps ?? [];
        $gapList = $gaps === [] ? 'No specific gaps were recorded.' : '- '.implode("\n- ", $gaps);

        return $this->complete(
            $this->draftSystemPrompt(),
            <<<PROMPT
            CANDIDATE NAME: {$application->candidate->name}
            ROLE: {$application->jobPosting->title}
            COMPANY: {$application->jobPosting->company->name}

            GAPS IDENTIFIED DURING SCREENING:
            {$gapList}
            PROMPT,
        );
    }

    public function enhance(Application $application, string $draft): string
    {
        $application->loadMissing(['candidate:id,name', 'jobPosting.company:id,name']);

        return $this->complete(
            $this->enhanceSystemPrompt(),
            <<<PROMPT
            CANDIDATE NAME: {$application->candidate->name}
            ROLE: {$application->jobPosting->title}
            COMPANY: {$application->jobPosting->company->name}

            THE HIRING TEAM WROTE:
            {$draft}
            PROMPT,
        );
    }

    private function draftSystemPrompt(): string
    {
        return <<<'PROMPT'
        You write rejection messages that a hiring team sends to a candidate.

        Be warm, direct, and specific. Reference the role by name. Mention at most two
        concrete reasons drawn only from the gaps you are given; never invent a reason
        and never mention a score, a ranking, or that AI was involved.

        Four to six sentences. No subject line, no placeholders such as [Name], and no
        promises about future openings you cannot keep. End with a brief good wish.
        PROMPT;
    }

    private function enhanceSystemPrompt(): string
    {
        return <<<'PROMPT'
        You rewrite a hiring team's rejection message so it reads clearly, warmly, and
        professionally.

        Keep every reason they gave and add none of your own. Do not change the decision
        or soften it into ambiguity. Do not mention scores, rankings, or AI. Return only
        the rewritten message, with no preamble and no placeholders.
        PROMPT;
    }

    private function complete(string $systemPrompt, string $userPrompt): string
    {
        $model = config('hr.ai.model');

        if (blank($model)) {
            throw new AiResponseException('OPENAI_MODEL is not set. Add it to your .env file.');
        }

        $response = OpenAI::responses()->create([
            'model' => $model,
            'input' => [
                ['role' => 'system', 'content' => $systemPrompt],
                ['role' => 'user', 'content' => $userPrompt],
            ],
        ]);

        $message = trim($response->outputText);

        if ($message === '') {
            throw new AiResponseException('The model returned an empty rejection message.');
        }

        return $message;
    }
}
```

- [ ] **Step 6: Write the decision actions**

```bash
php artisan make:class Actions/Applications/ShortlistApplication --no-interaction
php artisan make:class Actions/Applications/RejectApplication --no-interaction
```

`app/Actions/Applications/ShortlistApplication.php`:

```php
<?php

namespace App\Actions\Applications;

use App\Enums\ApplicationStatus;
use App\Models\Application;
use App\Models\User;

class ShortlistApplication
{
    /**
     * Move a candidate to the next step.
     */
    public function handle(User $decider, Application $application): Application
    {
        $application->forceFill([
            'status' => ApplicationStatus::Shortlisted,
            'rejection_reason' => null,
            'decided_at' => now(),
            'decided_by' => $decider->id,
        ])->save();

        return $application;
    }
}
```

`app/Actions/Applications/RejectApplication.php`:

```php
<?php

namespace App\Actions\Applications;

use App\Enums\ApplicationStatus;
use App\Models\Application;
use App\Models\User;

class RejectApplication
{
    /**
     * Reject a candidate and store the message they will see.
     */
    public function handle(User $decider, Application $application, string $reason): Application
    {
        $application->forceFill([
            'status' => ApplicationStatus::Rejected,
            'rejection_reason' => $reason,
            'decided_at' => now(),
            'decided_by' => $decider->id,
        ])->save();

        return $application;
    }
}
```

- [ ] **Step 7: Write the form requests**

```bash
php artisan make:request Hr/RejectApplicationRequest --no-interaction
php artisan make:request Hr/EnhanceRejectionRequest --no-interaction
```

`app/Http/Requests/Hr/RejectApplicationRequest.php`:

```php
<?php

namespace App\Http\Requests\Hr;

use Illuminate\Contracts\Validation\ValidationRule;
use Illuminate\Foundation\Http\FormRequest;

class RejectApplicationRequest extends FormRequest
{
    /**
     * @return array<string, ValidationRule|array<mixed>|string>
     */
    public function rules(): array
    {
        return [
            'rejection_reason' => ['required', 'string', 'min:20', 'max:2000'],
        ];
    }

    /**
     * @return array<string, string>
     */
    public function messages(): array
    {
        return [
            'rejection_reason.min' => __('Write at least a sentence explaining the decision.'),
        ];
    }
}
```

`app/Http/Requests/Hr/EnhanceRejectionRequest.php`:

```php
<?php

namespace App\Http\Requests\Hr;

use Illuminate\Contracts\Validation\ValidationRule;
use Illuminate\Foundation\Http\FormRequest;

class EnhanceRejectionRequest extends FormRequest
{
    /**
     * @return array<string, ValidationRule|array<mixed>|string>
     */
    public function rules(): array
    {
        return [
            'message' => ['required', 'string', 'max:2000'],
        ];
    }
}
```

- [ ] **Step 8: Write the controllers**

```bash
php artisan make:controller Hr/ApplicationDecisionController --no-interaction
php artisan make:controller Hr/RejectionMessageController --no-interaction
```

`app/Http/Controllers/Hr/ApplicationDecisionController.php`:

```php
<?php

namespace App\Http\Controllers\Hr;

use App\Actions\Applications\RejectApplication;
use App\Actions\Applications\ShortlistApplication;
use App\Http\Controllers\Controller;
use App\Http\Requests\Hr\RejectApplicationRequest;
use App\Models\Application;
use Illuminate\Http\RedirectResponse;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Gate;
use Inertia\Inertia;

class ApplicationDecisionController extends Controller
{
    /**
     * Advance a candidate to the next step.
     */
    public function shortlist(Request $request, Application $application, ShortlistApplication $shortlistApplication): RedirectResponse
    {
        Gate::authorize('review', $application);

        $shortlistApplication->handle($request->user(), $application);

        Inertia::flash('toast', ['type' => 'success', 'message' => __('Candidate shortlisted.')]);

        return back();
    }

    /**
     * Reject a candidate with the message they will read.
     */
    public function reject(RejectApplicationRequest $request, Application $application, RejectApplication $rejectApplication): RedirectResponse
    {
        Gate::authorize('review', $application);

        $rejectApplication->handle(
            $request->user(),
            $application,
            $request->string('rejection_reason')->toString(),
        );

        Inertia::flash('toast', ['type' => 'success', 'message' => __('Candidate rejected and notified in their portal.')]);

        return back();
    }
}
```

`app/Http/Controllers/Hr/RejectionMessageController.php`:

```php
<?php

namespace App\Http\Controllers\Hr;

use App\Contracts\RejectionWriter;
use App\Http\Controllers\Controller;
use App\Http\Requests\Hr\EnhanceRejectionRequest;
use App\Models\Application;
use Illuminate\Http\JsonResponse;
use Illuminate\Support\Facades\Gate;

class RejectionMessageController extends Controller
{
    /**
     * Write a rejection message from scratch.
     */
    public function draft(Application $application, RejectionWriter $writer): JsonResponse
    {
        Gate::authorize('review', $application);

        return response()->json(['message' => $writer->draft($application)]);
    }

    /**
     * Improve the message the hiring team already wrote.
     */
    public function enhance(EnhanceRejectionRequest $request, Application $application, RejectionWriter $writer): JsonResponse
    {
        Gate::authorize('review', $application);

        return response()->json([
            'message' => $writer->enhance($application, $request->string('message')->toString()),
        ]);
    }
}
```

These two return JSON rather than a redirect, which is what lets the React dialog stay open while the text is replaced.

- [ ] **Step 9: Bind the contract and add the routes**

In `app/Providers/AppServiceProvider::register()`:

```php
        $this->app->bind(\App\Contracts\RejectionWriter::class, \App\Services\Ai\OpenAiRejectionWriter::class);
```

Add inside the group in `routes/hr.php`:

```php
        Route::post('applications/{application}/shortlist', [ApplicationDecisionController::class, 'shortlist'])
            ->name('applications.shortlist');

        Route::post('applications/{application}/reject', [ApplicationDecisionController::class, 'reject'])
            ->name('applications.reject');

        Route::post('applications/{application}/rejection/draft', [RejectionMessageController::class, 'draft'])
            ->middleware('throttle:10,1')
            ->name('applications.rejection.draft');

        Route::post('applications/{application}/rejection/enhance', [RejectionMessageController::class, 'enhance'])
            ->middleware('throttle:10,1')
            ->name('applications.rejection.enhance');
```

with the imports:

```php
use App\Http\Controllers\Hr\ApplicationDecisionController;
use App\Http\Controllers\Hr\RejectionMessageController;
```

- [ ] **Step 10: Build the reject dialog**

`useHttp`'s `post()` resolves with the parsed JSON body, so the AI text arrives directly. The textarea is bound to `useHttp`'s own `data.message` — one source of truth for AI output and manual typing alike — and the real rejection goes through an Inertia `<Form>` so server validation errors render normally.

`resources/js/components/reject-dialog.tsx`:

```tsx
import { Form, useHttp } from '@inertiajs/react';
import { useState } from 'react';
import InputError from '@/components/input-error';
import { Button } from '@/components/ui/button';
import {
    Dialog,
    DialogContent,
    DialogDescription,
    DialogFooter,
    DialogHeader,
    DialogTitle,
    DialogTrigger,
} from '@/components/ui/dialog';
import { Label } from '@/components/ui/label';
import { Spinner } from '@/components/ui/spinner';
import { Textarea } from '@/components/ui/textarea';
import { reject } from '@/routes/hr/applications';
import { draft, enhance } from '@/routes/hr/applications/rejection';

type RejectionResponse = {
    message: string;
};

type Props = {
    applicationId: number;
    candidateName: string;
};

export default function RejectDialog({ applicationId, candidateName }: Props) {
    const [open, setOpen] = useState(false);

    const { data, setData, post, processing } = useHttp<
        { message: string },
        RejectionResponse
    >({ message: '' });

    const generate = async () => {
        const response = await post(draft.url(applicationId));
        setData('message', response.message);
    };

    const improve = async () => {
        const response = await post(enhance.url(applicationId));
        setData('message', response.message);
    };

    return (
        <Dialog open={open} onOpenChange={setOpen}>
            <DialogTrigger asChild>
                <Button variant="destructive" size="sm">
                    Reject
                </Button>
            </DialogTrigger>

            <DialogContent className="sm:max-w-2xl">
                <DialogHeader>
                    <DialogTitle>Reject {candidateName}</DialogTitle>
                    <DialogDescription>
                        Write the message yourself, or have AI draft it and edit before sending.
                        {candidateName} will read this on their applications page.
                    </DialogDescription>
                </DialogHeader>

                <Form
                    {...reject.form(applicationId)}
                    onSuccess={() => setOpen(false)}
                    className="flex flex-col gap-4"
                >
                    {({ processing: submitting, errors }) => (
                        <>
                            <div className="flex flex-wrap gap-2">
                                <Button
                                    type="button"
                                    variant="secondary"
                                    size="sm"
                                    onClick={generate}
                                    disabled={processing}
                                >
                                    {processing && <Spinner />}
                                    Generate with AI
                                </Button>
                                <Button
                                    type="button"
                                    variant="outline"
                                    size="sm"
                                    onClick={improve}
                                    disabled={processing || data.message.trim() === ''}
                                >
                                    {processing && <Spinner />}
                                    Enhance my text
                                </Button>
                            </div>

                            <div className="grid gap-2">
                                <Label htmlFor="rejection_reason">Message to the candidate</Label>
                                <Textarea
                                    id="rejection_reason"
                                    name="rejection_reason"
                                    rows={10}
                                    value={data.message}
                                    onChange={(event) => setData('message', event.target.value)}
                                    placeholder="Explain the decision in a sentence or two."
                                    required
                                />
                                <InputError message={errors.rejection_reason} />
                            </div>

                            <DialogFooter>
                                <Button type="button" variant="ghost" onClick={() => setOpen(false)}>
                                    Cancel
                                </Button>
                                <Button type="submit" variant="destructive" disabled={submitting || processing}>
                                    {submitting && <Spinner />}
                                    Reject candidate
                                </Button>
                            </DialogFooter>
                        </>
                    )}
                </Form>
            </DialogContent>
        </Dialog>
    );
}
```

- [ ] **Step 11: Add the decision controls to the application page**

In `resources/js/pages/hr/applications/show.tsx`, add the imports:

```tsx
import RejectDialog from '@/components/reject-dialog';
import { shortlist } from '@/routes/hr/applications';
```

and insert this block directly above `<ScreeningPanel …>`:

```tsx
            {application.status === 'submitted' && (
                <div className="flex flex-wrap items-center gap-2">
                    <Link href={shortlist(application.id)} method="post" as="button">
                        <Button size="sm">Move to next step</Button>
                    </Link>
                    <RejectDialog
                        applicationId={application.id}
                        candidateName={application.candidate_name}
                    />
                </div>
            )}

            {application.status !== 'submitted' && application.decided_by && (
                <p className="text-sm text-muted-foreground">
                    Decided by {application.decided_by}.
                </p>
            )}
```

- [ ] **Step 12: Regenerate routes and run the tests**

```bash
php artisan wayfinder:generate --with-form --no-interaction
php artisan test --compact --filter=DecideApplicationTest
```

Expected: PASS (9 tests).

- [ ] **Step 13: Format and commit**

```bash
vendor/bin/pint --dirty --format agent
git add -A
git commit -m "feat: shortlist and reject candidates with ai written messages"
```

---

## Task 9: Demo data and full verification

**Files:**
- Create: `database/seeders/DemoSeeder.php`
- Modify: `database/seeders/DatabaseSeeder.php`
- Test: the whole suite

**Interfaces:**
- Consumes: every factory and seeder from Tasks 1 through 8
- Produces: `DemoSeeder`, which creates one company, an admin, an HR user, three postings and eight applications with completed screenings

- [ ] **Step 1: Write the demo seeder**

```bash
php artisan make:seeder DemoSeeder --no-interaction
```

`database/seeders/DemoSeeder.php`:

```php
<?php

namespace Database\Seeders;

use App\Enums\RoleName;
use App\Models\Application;
use App\Models\ApplicationScreening;
use App\Models\Company;
use App\Models\JobPosting;
use App\Models\User;
use Illuminate\Database\Seeder;
use Illuminate\Support\Facades\Hash;

class DemoSeeder extends Seeder
{
    /**
     * Populate a browsable demo workspace. Local use only.
     */
    public function run(): void
    {
        $company = Company::factory()->create([
            'name' => 'Nimbus Labs',
            'slug' => 'nimbus-labs',
        ]);

        $admin = User::factory()->create([
            'name' => 'Dana Okafor',
            'email' => 'admin@nimbus.test',
            'password' => Hash::make('password'),
            'company_id' => $company->id,
        ]);
        $admin->assignRole(RoleName::CompanyAdmin->value);

        $recruiter = User::factory()->create([
            'name' => 'Sam Haddad',
            'email' => 'hr@nimbus.test',
            'password' => Hash::make('password'),
            'company_id' => $company->id,
        ]);
        $recruiter->assignRole(RoleName::Hr->value);

        $postings = collect([
            'Senior Backend Engineer',
            'Product Designer',
            'Data Analyst',
        ])->map(fn (string $title): JobPosting => JobPosting::factory()->published()->create([
            'company_id' => $company->id,
            'created_by' => $admin->id,
            'title' => $title,
        ]));

        $candidates = User::factory()->count(8)->create([
            'password' => Hash::make('password'),
        ]);

        $candidates->each(function (User $candidate) use ($postings): void {
            $candidate->assignRole(RoleName::Candidate->value);

            $application = Application::factory()->create([
                'job_posting_id' => $postings->random()->id,
                'user_id' => $candidate->id,
            ]);

            ApplicationScreening::factory()->for($application)->completed()->create();
        });

        $this->command->info('Demo login: admin@nimbus.test / password');
        $this->command->info('Demo login: hr@nimbus.test / password');
    }
}
```

The seeded applications carry a fake `cv_path`, so downloading a demo CV will 404. That is expected — apply through the UI with a real PDF to exercise the full pipeline.

- [ ] **Step 2: Wire it behind an explicit call**

In `database/seeders/DatabaseSeeder.php`:

```php
    public function run(): void
    {
        $this->call(RolesAndPermissionsSeeder::class);

        if (app()->environment('local')) {
            $this->call(DemoSeeder::class);
        }
    }
```

- [ ] **Step 3: Reseed and eyeball it**

```bash
php artisan migrate:fresh --seed
```

- [ ] **Step 4: Run the whole suite**

```bash
php artisan test --compact
```

Expected: PASS across every file. If `ExampleTest.php` in `tests/Feature` still asserts the old welcome page, leave it — it should still pass.

- [ ] **Step 5: Run the static and style checks**

```bash
vendor/bin/pint --format agent
composer types:check
npm run types:check
npm run lint
```

Fix anything they report. `types:check` runs Larastan; if it flags `$request->user()` as possibly null in HR controllers, add `/** @var \App\Models\User $user */` above the assignment rather than weakening the type.

- [ ] **Step 6: Build the front end**

```bash
npm run build
```

- [ ] **Step 7: Walk the flow end to end**

Run both processes:

```bash
composer run dev
```

Then, in a browser:

1. Register a company at `/register/company`.
2. Create a **published** posting.
3. Invite a teammate at `/hr/team`, copy the link, open it in a private window, accept it.
4. Register a candidate at `/register`, browse `/jobs`, open a posting, upload a real PDF CV.
5. Confirm the queue worker picks up `ScreenApplicationJob` and the score appears on `/hr/jobs/{id}/applications` within a few seconds.
6. Open a candidate, click **Generate with AI**, edit the text, click **Enhance my text**, then reject.
7. Log back in as the candidate and confirm the message shows on `/my/applications`.

If step 5 never resolves, check that a worker is running and that `OPENAI_MODEL` names a model your key can reach.

- [ ] **Step 8: Commit**

```bash
git add -A
git commit -m "feat: add demo seeder and finish hr assistant mvp"
```

---

## Appendix: Troubleshooting

**"There is no role named `company_admin`"** — the roles seeder did not run for that test. Confirm the `beforeEach` hook in `tests/Pest.php` from Task 1 Step 10 is present.

**Wayfinder import not found** — run `php artisan wayfinder:generate --with-form --no-interaction`. The Vite plugin regenerates on `npm run dev`, but a route added while Vite is stopped needs the manual run.

**`Unable to locate file in Vite manifest`** — run `npm run build`, or start `npm run dev`.

**Screening stays "pending" forever** — no queue worker. Run `php artisan queue:work`. Check failures with `php artisan queue:failed`.

**`Target [App\Contracts\CvScreener] is not instantiable`** — the binding in `AppServiceProvider::register()` from Task 6 Step 8 is missing.

**Radix `<Select>` submits nothing** — expected. It renders no native form control. Use a native `<select>` inside Inertia's uncontrolled `<Form>`, as `job-posting-form.tsx` does.

**`OpenAI::fake()` assertions fail on payload shape** — the Responses API fixture shape can shift between client versions. The installed defaults live in `vendor/openai-php/client/src/Testing/Responses/Fixtures/Responses/CreateResponseFixture.php`. Match your fake payload to that file. The suite does not depend on this: every behavioural test uses `FakeCvScreener` / `FakeRejectionWriter` via `swap()`.

---

## Self-Review

**Spec coverage** — every section of the spec maps to a task:

| Spec section | Task |
| --- | --- |
| 2 New dependencies | 0 |
| 4 Data model, enums | 1 |
| 5 Roles, policies, account creation | 1, 2, 3, 4, 7 |
| 6 Application architecture | 3, 5, 6, 8 |
| 7 Screening pipeline, structured output | 6 |
| 8 Decision flow, draft/enhance | 8 |
| 9 Files and security | 5, 7 |
| 10 Routes | 2, 3, 4, 5, 7, 8 |
| 11 Front end | 3, 4, 5, 7, 8 |
| 12 Testing | every task |
| 13 Build order | task order |

**Deviations from the spec, deliberate:**

- The spec listed a single `hr/team/index.tsx`; the plan uses `hr/team.tsx` because there is only one team screen and Inertia resolves the flatter name fine.
- The spec did not name `PermissionName`; it was added so policies do not import permission strings from a seeder.
- The spec's `RescreenController` was implied by "a Re-run screening button"; it is an explicit route here.

**Known ordering constraint:** `tests/Feature/DashboardTest.php` is rewritten in Task 2 but cannot pass until Task 5. Both tasks call this out. Every other task is green on its own.

