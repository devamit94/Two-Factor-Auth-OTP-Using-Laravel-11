# Two-Factor-Auth-OTP-Using-Laravel-11
Set, verification code and iys expiration time, add two fields to the default users migration:

database/migrations/0001_01_01_000000_create_users_table.php

    public function up(): void
    {
        Schema::create('users', function (Blueprint $table) {
            $table->id();
            $table->string('name');
            $table->string('email')->unique();
            $table->timestamp('email_verified_at')->nullable();
            $table->string('password');
            $table->rememberToken();
            $table->string('two_factor_code')->nullable();
            $table->dateTime('two_factor_expires_at')->nullable();
            $table->timestamps();
        });

        Schema::create('password_reset_tokens', function (Blueprint $table) {
            $table->string('email')->primary();
            $table->string('token');
            $table->timestamp('created_at')->nullable();
        });

        Schema::create('sessions', function (Blueprint $table) {
            $table->string('id')->primary();
            $table->foreignId('user_id')->nullable()->index();
            $table->string('ip_address', 45)->nullable();
            $table->text('user_agent')->nullable();
            $table->longText('payload');
            $table->integer('last_activity')->index();
        });
    }
Add those fields to app/Models/User.php properties $fillable array:

    protected $fillable = [
        'name',
        'email',
        'password',
        'two_factor_code',
        'two_factor_expires_at',
    ];
For filling those fields let's create a method in the User model: app/Models/User.php:

public function generateTwoFactorCode()
{
    $this->timestamp = false;
    $this->two_factor_code = rand(100000, 999999);
    $this->two_factor_expires_at = now()->addMinutes(10);
}
Notice: in addition to creating a random code and setting its expiration time, we also specify that this update should not touch the updated_at column in users table – so we’re doing $this->timestamps = false;.

Sending Code via Email
Now generate the code and send it to the user. For that, let's generate a Laravel Notification class:

php artisan make:notification SendTwoFactorCode
The main benefit of using Notifications is that we can provide the channel(s) how to send that notification: email, SMS and others.

app/Notifications/SendTwoFactorCode.php:

class SendTwoFactorCode extends Notification
{
    public function toMail(object $notifiable): MailMessage
    {
        return (new MailMessage)
                    ->line('Your two factor code is {$notifiable->two_factor_code}')
                    ->action('Verify here Action', route('verify.index'))
                    ->line('The code will expire in 10 minutes')
                    ->line('If you have not tried to login, ignore this message.');
    }
}
We will create the route('verify.index') route, which will also re-send the code, a bit later.

As you can see, we use the $notifiable variable, which automatically represents the recipient of the notification - in our case, the logged-in user themselves.

Now, how/where do we call that notification?

We're using Laravel Breeze, so after login, in the app/Http/Controllers/Auth/AuthenticatedSessionController.php, we need to add this code in the store() method:

    public function store(LoginRequest $request): RedirectResponse
    {
        $request->authenticate();

        $request->session()->regenerate();

        $request->user()->generateTwoFactorCode();

        $request->user()->notify(new SendTwoFactorCode());

        return redirect()->intended(route('dashboard', absolute: false));
    }
This will send an email which will look like this:

Redirecting to Verification Form Let's build the restriction middleware. Until the logged-in user enters the verification code, they need to be redirected to this form:

verification form

To perform that restriction, we will generate a Middleware:

php artisan make:middleware TwoFactorMiddleware

app/Http/Middleware/TwoFactorMiddleware.php:

class TwoFactorMiddleware
{
    public function handle(Request $request, Closure $next): Response
    {
        $user = auth()->user();

        if (auth()->check() && $user->two_factor_code) {
            if ($user->two_factor_expires_at < now()) {
                $user->resetTwoFactorCode();
                auth()->logout();

                return redirect()->route('login')
                    ->withStatus(__('The two factor code has expired. Please login again.'));
            }

            if (!$request->is('verify.*')){
                return redirect()->route('verify.index');
            }
        }
        return $next($request);
    }
}
If you're not familiar with how Middleware works, read the official Laravel documentation. But basically, it’s a class that performs some actions to usually restrict accessing some page or function.

So, in our case, we check if there is a two-factor code set. If it is, we check if it isn't expired. If it is expired, we reset it and redirect back to the login form. If it's still active, we redirect back to the verification form.

There is a new method resetTwoFactorCode(). Add it to the app/Models/User.php, it should look like this:

public function resetTwoFactorCode(): void
{
    $this->timestamps = false;
    $this->two_factor_code = null;
    $this->two_factor_expires_at = null;
    $this->save();
}
Next, we need to register our middleware in bootstrap/app.php by giving an "alias" name, let's call it twofactor:

<?php

use App\Http\Middleware\TwoFactorMiddleware;
use Illuminate\Foundation\Application;
use Illuminate\Foundation\Configuration\Exceptions;
use Illuminate\Foundation\Configuration\Middleware;

return Application::configure(basePath: dirname(__DIR__))
    ->withRouting(
        web: __DIR__.'/../routes/web.php',
        commands: __DIR__.'/../routes/console.php',
        health: '/up',
    )
    ->withMiddleware(function (Middleware $middleware) {
        $middleware->alias([
            'twofactor' => TwoFactorMiddleware::class,
           
        ]);
    })
    ->withExceptions(function (Exceptions $exceptions) {
        //
    })->create();

Now, we can assign our twofactor Middleware to the routes in routes/web.php. Since Laravel Breeze has only one protected route /dashboard, we will add our Middleware to that route:

Route::get('/dashboard', function () {
    return view('dashboard');
})->middleware(['auth', 'twofactor'])->name('dashboard');
Now, our dashboard is protected by two-factor authentication.

Verification Page Controller & View At this point, any request to any URL will redirect to the code verification page. Time to build it - both the page, and its processing method.

We start with generating the Controller:

php artisan make:controller TwoFactorController

Then, we add three routes leading to the new Controller:

routes/web.php:

Route::middleware(['auth', 'twofactor'])->group(function () {
    Route::get('verify/resend', [TwoFactorController::class, 'resend'])->name('verify.resend');
    Route::resource('verify', TwoFactorController::class)->only(['index', 'store']);
});
And all the logic in the Controller looks like this.

app/Http/Controllers/Auth/TwoFactorController:

class TwoFactorController extends Controller
{
    public function index(): View
    {
        return view('auth.twoFactor');
    }
 
    public function store(Request $request): ValidationException|RedirectResponse
    {
        $request->validate([
            'two_factor_code' => ['integer', 'required'],
        ]);
 
        $user = auth()->user();
 
        if ($request->input('two_factor_code') !== $user->two_factor_code) {
            throw ValidationException::withMessages([
                'two_factor_code' => __('The two factor code you have entered does not match'),
            ]);
        }
 
        $user->resetTwoFactorCode();
 
        return redirect()->to(RouteServiceProvider::HOME);
    }
 
    public function resend(): RedirectResponse
    {
        $user = auth()->user();
        $user->generateTwoFactorCode();
        $user->notify(new SendTwoFactorCode());
 
        return redirect()->back()->withStatus(__('The two factor code has been sent again'));
    }
}
Three methods here:

index() method just shows the form. Then, this form is submitted with POST request to the store() method to verify the code And lastly, resend() method re-generates and re-sends new code. The form itself just uses the default layout and components from Laravel Breeze, the code looks like this:

resources/views/auth/twoFactor.blade.php:

app/Notifications/SendTwoFactorCode.php:

About
Laravel: Two Factor Auth OTP via Email and SMS

Resources
 Readme
 Activity
Stars
 0 stars
Watchers
 1 watching
Forks
 0 forks
Report repository
Releases
No releases published
Packages
No packages published
Languages
PHP
52.2%
 
Blade
47.1%
 
Other
0.7%
Footer
© 2025 GitHub, Inc.
Footer navigation
Terms
Privacy
Security
Status
Docs
Contact
