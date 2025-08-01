# Tes-Teori-Solopos

### 1\. Jelaskan perbedaan antara Service Container dan Service Provider di Laravel. Bagaimana cara menambahkan binding custom ke Service Container?

**Service Container** adalah alat yang kuat untuk mengelola dependensi kelas dan melakukan injeksi dependensi. Secara sederhana, ini adalah "wadah" di mana Laravel menyimpan dan mengelola semua kelas yang dibutuhkan aplikasi Anda. Ketika Anda membutuhkan sebuah kelas, Service Container akan secara otomatis menyelesaikan dependensi kelas tersebut, termasuk dependensi bersarang.

**Service Provider** adalah tempat di mana semua pengikatan (binding) Service Container didaftarkan. Mereka adalah jembatan antara Service Container dan kode aplikasi Anda. Service Provider bertanggung jawab untuk "memberitahu" Laravel bagaimana cara membangun berbagai komponen aplikasi, seperti layanan, repository, atau helper khusus.

**Perbedaan Utama:**

  * **Service Container:** Mekanisme untuk mengelola dan menyelesaikan dependensi.
  * **Service Provider:** Tempat untuk mendaftarkan dan mengkonfigurasi binding ke Service Container.

**Cara menambahkan binding custom ke Service Container:**

Anda dapat menambahkan binding custom di metode `register()` dari Service Provider. Ada beberapa cara:

  * **Basic Binding:** Mengikat implementasi kelas ke sebuah interface.

    ```php
    $this->app->bind(
        'App\Contracts\MyContract',
        'App\Services\MyService'
    );
    ```

  * **Singleton Binding:** Mengikat kelas sebagai singleton (hanya satu instance yang dibuat).

    ```php
    $this->app->singleton(
        'App\Contracts\MyContract',
        'App\Services\MyService'
    );
    ```

  * **Instance Binding:** Mengikat instance objek yang sudah ada.

    ```php
    $myService = new App\Services\MyService();
    $this->app->instance(
        'App\Contracts\MyContract',
        $myService
    );
    ```

  * **Contextual Binding:** Mengikat implementasi yang berbeda tergantung pada kelas yang membutuhkan.

    ```php
    $this->app->when('App\Http\Controllers\UserController')
              ->needs('App\Contracts\MyContract')
              ->give('App\Services\SpecificUserService');
    ```

Biasanya, Anda akan membuat Service Provider baru (`php artisan make:provider MyServiceProvider`) dan mendaftarkannya di `config/app.php`.

### 2\. Apa itu Repository Pattern? Implementasikan dengan contoh sederhana (tanpa package) untuk entitas Post.

**Repository Pattern** adalah pola desain yang mengisolasi lapisan data dari lapisan bisnis aplikasi. Ini menyediakan abstraksi untuk operasi penyimpanan data, sehingga lapisan bisnis tidak perlu mengetahui bagaimana data diambil atau disimpan. Ini mempromosikan kode yang lebih bersih, dapat diuji, dan lebih mudah dikelola karena Anda dapat mengganti implementasi penyimpanan data tanpa memengaruhi logika bisnis.

**Implementasi sederhana untuk entitas Post (tanpa package):**

**1. Interface Repository (`app/Repositories/PostRepository.php`)**

```php
<?php

namespace App\Repositories;

use App\Models\Post;
use Illuminate\Database\Eloquent\Collection;

interface PostRepository
{
    public function getAll(): Collection;
    public function findById(int $id): ?Post;
    public function create(array $data): Post;
    public function update(int $id, array $data): bool;
    public function delete(int $id): bool;
}
```

**2. Implementasi Repository Eloquent (`app/Repositories/EloquentPostRepository.php`)**

```php
<?php

namespace App\Repositories;

use App\Models\Post;
use Illuminate\Database\Eloquent\Collection;

class EloquentPostRepository implements PostRepository
{
    public function getAll(): Collection
    {
        return Post::all();
    }

    public function findById(int $id): ?Post
    {
        return Post::find($id);
    }

    public function create(array $data): Post
    {
        return Post::create($data);
    }

    public function update(int $id, array $data): bool
    {
        $post = Post::find($id);
        if ($post) {
            return $post->update($data);
        }
        return false;
    }

    public function delete(int $id): bool
    {
        $post = Post::find($id);
        if ($post) {
            return $post->delete();
        }
        return false;
    }
}
```

**3. Model Post (`app/Models/Post.php`)**

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Model;

class Post extends Model
{
    use HasFactory;

    protected $fillable = ['title', 'content', 'author_id'];
}
```

**4. Binding di Service Provider (`app/Providers/RepositoryServiceProvider.php`)**

```php
<?php

namespace App\Providers;

use App\Repositories\EloquentPostRepository;
use App\Repositories\PostRepository;
use Illuminate\Support\ServiceProvider;

class RepositoryServiceProvider extends ServiceProvider
{
    public function register()
    {
        $this->app->bind(PostRepository::class, EloquentPostRepository::class);
    }

    public function boot()
    {
        //
    }
}
```

Jangan lupa daftarkan `RepositoryServiceProvider` di `config/app.php` di bagian `providers`.

**5. Penggunaan di Controller (`app/Http/Controllers/PostController.php`)**

```php
<?php

namespace App\Http\Controllers;

use App\Repositories\PostRepository;
use Illuminate\Http\Request;

class PostController extends Controller
{
    protected $postRepository;

    public function __construct(PostRepository $postRepository)
    {
        $this->postRepository = $postRepository;
    }

    public function index()
    {
        $posts = $this->postRepository->getAll();
        return view('posts.index', compact('posts'));
    }

    public function show(int $id)
    {
        $post = $this->postRepository->findById($id);
        if (!$post) {
            abort(404);
        }
        return view('posts.show', compact('post'));
    }

    public function store(Request $request)
    {
        $post = $this->postRepository->create($request->all());
        return redirect()->route('posts.show', $post->id);
    }

    // ... method lain seperti update dan delete
}
```

### 3\. Jelaskan bagaimana Lazy Loading dan Eager Loading bekerja di Eloquent. Berikan contoh situasi nyata dan dampaknya terhadap performa.

Di Eloquent, **Lazy Loading** dan **Eager Loading** berkaitan dengan cara memuat relasi antar model.

**Lazy Loading (Pemuatan Malas):**

  * **Cara Kerja:** Relasi tidak dimuat sampai Anda secara eksplisit mengaksesnya. Setiap kali Anda mengakses relasi, Eloquent akan menjalankan query database baru untuk mengambil data relasi tersebut.
  * **Contoh:**
    Misalkan Anda memiliki model `User` yang memiliki banyak `Post`.
    ```php
    $users = App\Models\User::all(); // Query: SELECT * FROM users
    foreach ($users as $user) {
        echo $user->name;
        echo $user->posts->count(); // Query: SELECT * FROM posts WHERE user_id = ? (untuk setiap user)
    }
    ```
    Dalam contoh ini, jika ada 1000 user, Eloquent akan menjalankan 1 query untuk mengambil semua user, dan kemudian 1000 query lagi untuk mengambil post dari masing-masing user. Ini dikenal sebagai masalah "N+1 query".
  * **Dampak Terhadap Performa:** Buruk untuk performa jika Anda perlu mengakses relasi untuk banyak model, karena akan menghasilkan banyak query database yang terpisah.

**Eager Loading (Pemuatan Cepat):**

  * **Cara Kerja:** Relasi dimuat bersamaan dengan model utama, biasanya dalam satu atau dua query tambahan (bukan N query terpisah). Ini dilakukan dengan menggunakan metode `with()`.
  * **Contoh:**
    Melanjutkan contoh `User` dan `Post`:
    ```php
    $users = App\Models\User::with('posts')->get(); // Query 1: SELECT * FROM users; Query 2: SELECT * FROM posts WHERE user_id IN (id_user_1, id_user_2, ...)
    foreach ($users as $user) {
        echo $user->name;
        echo $user->posts->count(); // Relasi sudah dimuat, tidak ada query baru
    }
    ```
    Dalam contoh ini, Eloquent hanya akan menjalankan 2 query: satu untuk user dan satu untuk semua post yang terkait dengan user-user tersebut.
  * **Dampak Terhadap Performa:** Sangat meningkatkan performa jika Anda perlu mengakses relasi untuk banyak model, karena mengurangi jumlah total query database secara drastis.

**Situasi Nyata dan Dampak:**

  * **Lazy Loading cocok untuk:** Situasi di mana Anda mungkin tidak selalu membutuhkan data relasi, atau hanya membutuhkan data relasi untuk satu atau beberapa instance model saja. Misalnya, menampilkan detail satu postingan dan mungkin beberapa komentar terkait, bukan daftar semua postingan dengan semua komentar mereka.
  * **Eager Loading cocok untuk:** Situasi di mana Anda pasti akan menggunakan data relasi untuk setiap model dalam sebuah koleksi. Contoh klasik adalah menampilkan daftar artikel di blog dan Anda juga perlu menampilkan nama penulis atau jumlah komentar untuk setiap artikel. Menggunakan eager loading akan mencegah "N+1 problem" dan membuat aplikasi Anda jauh lebih cepat.

### 4\. Laravel menyediakan fitur Job dan Queue. Jelaskan bagaimana kamu akan memproses pengiriman email massal menggunakan fitur ini. Sertakan struktur kode atau pseudocode.

**Job dan Queue** di Laravel digunakan untuk menunda eksekusi tugas yang memakan waktu (seperti pengiriman email, pemrosesan gambar, dll.) ke latar belakang. Ini membuat respons aplikasi tetap cepat bagi pengguna.

**Pengiriman Email Massal Menggunakan Job dan Queue:**

Untuk mengirim email massal, kita tidak ingin proses pengiriman email memblokir permintaan HTTP (misalnya, setelah pengguna mengklik tombol "kirim"). Dengan Job dan Queue, setiap email atau batch email dapat dienkapsulasi dalam sebuah "Job" dan didorong ke "Queue" untuk diproses secara asinkron oleh worker terpisah.

**Langkah-langkah:**

1.  **Buat Mailable:** Representasi email yang akan dikirim.

    ```bash
    php artisan make:mail MassEmail
    ```

    `app/Mail/MassEmail.php`

    ```php
    <?php

    namespace App\Mail;

    use Illuminate\Bus\Queueable;
    use Illuminate\Contracts\Queue\ShouldQueue;
    use Illuminate\Mail\Mailable;
    use Illuminate\Mail\Mailables\Content;
    use Illuminate\Mail\Mailables\Envelope;
    use Illuminate\Queue\SerializesModels;

    class MassEmail extends Mailable
    {
        use Queueable, SerializesModels;

        public $details;

        public function __construct($details)
        {
            $this->details = $details;
        }

        public function envelope(): Envelope
        {
            return new Envelope(
                subject: 'Pemberitahuan Penting: ' . $this->details['subject'],
            );
        }

        public function content(): Content
        {
            return new Content(
                view: 'emails.mass-email', // View Blade untuk konten email
            );
        }
    }
    ```

    `resources/views/emails/mass-email.blade.php` (contoh)

    ```blade
    <h1>{{ $details['subject'] }}</h1>
    <p>{{ $details['body'] }}</p>
    ```

2.  **Buat Job:** Kelas yang akan berisi logika pengiriman email. Job ini akan mengimplementasikan `ShouldQueue`.

    ```bash
    php artisan make:job SendMassEmail
    ```

    `app/Jobs/SendMassEmail.php`

    ```php
    <?php

    namespace App\Jobs;

    use App\Mail\MassEmail;
    use Illuminate\Bus\Queueable;
    use Illuminate\Contracts\Queue\ShouldQueue;
    use Illuminate\Foundation\Bus\Dispatchable;
    use Illuminate\Queue\InteractsWithQueue;
    use Illuminate\Queue\SerializesModels;
    use Illuminate\Support\Facades\Mail;

    class SendMassEmail implements ShouldQueue
    {
        use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

        protected $recipientEmail;
        protected $emailDetails;

        public function __construct(string $recipientEmail, array $emailDetails)
        {
            $this->recipientEmail = $recipientEmail;
            $this->emailDetails = $emailDetails;
        }

        public function handle(): void
        {
            Mail::to($this->recipientEmail)->send(new MassEmail($this->emailDetails));
        }
    }
    ```

3.  **Dispatch Job:** Dari controller atau lokasi lain, Anda mendispatch job ini.

    ```php
    // Misalnya di sebuah Controller
    use App\Jobs\SendMassEmail;
    use App\Models\User; // Misalkan daftar penerima dari model User

    class EmailController extends Controller
    {
        public function sendMassEmails(Request $request)
        {
            $users = User::all(); // Ambil semua user
            $emailDetails = [
                'subject' => $request->input('subject'),
                'body' => $request->input('body'),
            ];

            foreach ($users as $user) {
                // Dispatch job untuk setiap user
                SendMassEmail::dispatch($user->email, $emailDetails);

                // Atau jika ingin menunda pengiriman
                // SendMassEmail::dispatch($user->email, $emailDetails)->delay(now()->addMinutes(5));
            }

            return back()->with('success', 'Email massal sedang dalam antrian pengiriman.');
        }
    }
    ```

4.  **Konfigurasi Queue Driver:** Di `.env`, atur `QUEUE_CONNECTION` (misalnya `redis` atau `database`). Jika `database`, jalankan migrasi `php artisan queue:table` dan `php artisan migrate`.

5.  **Jalankan Queue Worker:** Untuk memproses job yang ada di queue.

    ```bash
    php artisan queue:work
    ```

    Untuk production, gunakan Supervisor atau systemd untuk menjaga worker tetap berjalan.

**Pseudocode (Flow Umum):**

```
FUNGSI kirimEmailMassal(subjek, isiEmail)
    DAFTAR_PENERIMA = ambilSemuaPenggunaDariDatabase()

    UNTUK setiap PENERIMA di DAFTAR_PENERIMA
        BUAT JobKirimEmailBaru(PENERIMA.email, subjek, isiEmail)
        MASUKKAN JobKirimEmailBaru KE Antrian

    KEMBALIKAN "Email sedang diproses di latar belakang."
AKHIR FUNGSI

// Proses di Background Worker
PROSES queue:work
    SEMENTARA AntrianTIDAKKosong
        AMBIL JobBerikutnyaDariAntrian
        EKSEKUSI JobBerikutnya
            // Misalnya, JobKirimEmail:
            // Kirim email ke PENERIMA menggunakan subjek dan isiEmail
            // Catat log jika berhasil/gagal
```

### 5\. Bagaimana kamu mengamankan endpoint API Laravel? Jelaskan konsep token-based authentication dan contoh implementasinya dengan Sanctum atau Passport.

Mengamankan endpoint API Laravel sangat penting untuk melindungi data dan memastikan hanya pengguna yang terautentikasi dan terotorisasi yang dapat mengakses sumber daya.

**Konsep Token-Based Authentication:**
Token-based authentication adalah metode autentikasi stateless di mana server tidak menyimpan informasi sesi pengguna. Sebaliknya, setiap permintaan dari klien disertai dengan token (biasanya JWT - JSON Web Token atau opaque token) yang membuktikan identitas klien.

**Flow umum:**

1.  **Login:** Klien mengirimkan kredensial (username/password) ke server.
2.  **Verifikasi & Token Generation:** Server memverifikasi kredensial. Jika valid, server menghasilkan token unik.
3.  **Token Issuance:** Server mengirimkan token kembali ke klien.
4.  **Subsequent Requests:** Klien menyimpan token (misalnya di `localStorage` atau `secure storage`) dan menyertakannya dalam header `Authorization` (biasanya sebagai `Bearer Token`) untuk setiap permintaan berikutnya ke API yang dilindungi.
5.  **Token Validation:** Server menerima permintaan, memvalidasi token (memastikan itu valid, tidak kedaluwarsa, dan tidak diubah), dan jika valid, memproses permintaan. Jika tidak valid, permintaan ditolak.

**Contoh Implementasi dengan Laravel Sanctum:**

Sanctum adalah solusi otentikasi lightweight untuk SPA (Single Page Applications), mobile applications, dan API berbasis token sederhana.

**1. Instalasi:**

```bash
composer require laravel/sanctum
php artisan vendor:publish --provider="Laravel\Sanctum\SanctumServiceProvider"
php artisan migrate
```

**2. Konfigurasi Model User:**
Tambahkan trait `HasApiTokens` ke model `User` Anda.

```php
// app/Models/User.php
<?php

namespace App\Models;

use Illuminate\Contracts\Auth\MustVerifyEmail;
use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Foundation\Auth\User as Authenticatable;
use Illuminate\Notifications\Notifiable;
use Laravel\Sanctum\HasApiTokens; // Import this

class User extends Authenticatable
{
    use HasFactory, Notifiable, HasApiTokens; // Add HasApiTokens
    // ...
}
```

**3. Autentikasi Login (Mendapatkan Token):**
Buat endpoint `/api/login` (atau serupa).

```php
// app/Http/Controllers/Api/AuthController.php
<?php

namespace App\Http\Controllers\Api;

use App\Http\Controllers\Controller;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Auth;
use Illuminate\Support\Facades\Hash;
use App\Models\User;

class AuthController extends Controller
{
    public function login(Request $request)
    {
        $request->validate([
            'email' => 'required|email',
            'password' => 'required',
        ]);

        $user = User::where('email', $request->email)->first();

        if (!$user || !Hash::check($request->password, $user->password)) {
            return response()->json(['message' => 'Kredensial tidak valid'], 401);
        }

        // Buat token. Anda bisa memberi nama token atau menentukan kemampuan (abilities)
        $token = $user->createToken('auth_token')->plainTextToken;

        return response()->json([
            'message' => 'Login berhasil',
            'access_token' => $token,
            'token_type' => 'Bearer',
        ]);
    }

    public function logout(Request $request)
    {
        // Menghapus token yang digunakan saat ini
        $request->user()->currentAccessToken()->delete();

        return response()->json(['message' => 'Logout berhasil']);
    }

    public function user(Request $request)
    {
        return response()->json($request->user());
    }
}
```

**4. Menggunakan Middleware `auth:sanctum`:**
Lindungi endpoint API Anda menggunakan middleware `auth:sanctum`.

```php
// routes/api.php
use App\Http\Controllers\Api\AuthController;
use App\Http\Controllers\Api\PostController; // Contoh controller lain

Route::post('/login', [AuthController::class, 'login']);

Route::middleware('auth:sanctum')->group(function () {
    Route::post('/logout', [AuthController::class, 'logout']);
    Route::get('/user', [AuthController::class, 'user']);

    Route::apiResource('posts', PostController::class);
});
```

**5. Mengirim Permintaan dari Klien:**
Klien harus menyertakan token yang diterima di header `Authorization`.

```
GET /api/user
Host: example.com
Authorization: Bearer YOUR_GENERATED_TOKEN_HERE
```

Dengan Sanctum, setiap kali klien mengirimkan token yang valid di header `Authorization`, Laravel akan secara otomatis mengautentikasi pengguna tersebut.

### 6\. Jika ada dua middleware yang harus dijalankan berurutan (CheckSubscription dan VerifyEmail), bagaimana Laravel mengatur urutannya? Apakah bisa mengubah urutan secara fleksibel?

Laravel mengatur urutan middleware berdasarkan dua hal:

1.  **Urutan Registrasi di `app/Http/Kernel.php`:**

      * **`$middleware` array:** Middleware yang tercantum di sini akan berjalan untuk setiap permintaan HTTP secara global, dalam urutan mereka didefinisikan.
      * **`$middlewareGroups` array:** Middleware dalam grup ini (misalnya `web` atau `api`) akan dijalankan dalam urutan yang ditentukan dalam grup tersebut.
      * **`$routeMiddleware` array:** Middleware individual yang diberi alias dan dapat diterapkan ke rute atau grup rute tertentu. Urutan di sini tidak langsung menentukan urutan eksekusi antar alias, melainkan urutan eksekusi *dalam satu alias* jika alias tersebut adalah grup middleware.

2.  **Urutan Penerapan pada Rute:**
    Ketika Anda menerapkan beberapa middleware ke rute (atau grup rute):

    ```php
    Route::middleware(['CheckSubscription', 'VerifyEmail'])->group(function () {
        // Rute-rute yang dilindungi
    });
    ```

    Middleware akan dijalankan dalam urutan yang **didefinisikan dalam array ini**, dari kiri ke kanan. Jadi, `CheckSubscription` akan berjalan terlebih dahulu, diikuti oleh `VerifyEmail`.

**Contoh Kasus `CheckSubscription` dan `VerifyEmail`:**

Jika Anda memiliki `CheckSubscription` dan `VerifyEmail` sebagai alias di `$routeMiddleware`:

```php
// app/Http/Kernel.php
protected $routeMiddleware = [
    // ...
    'CheckSubscription' => \App\Http\Middleware\CheckSubscription::class,
    'VerifyEmail' => \App\Http\Middleware\VerifyEmail::class,
];
```

Dan Anda menerapkannya pada rute seperti ini:

```php
Route::middleware(['CheckSubscription', 'VerifyEmail'])->group(function () {
    Route::get('/dashboard', [DashboardController::class, 'index']);
});
```

Maka:

1.  Permintaan masuk akan melewati `CheckSubscription`.
2.  Jika `CheckSubscription` lolos, permintaan akan diteruskan ke `VerifyEmail`.
3.  Jika `VerifyEmail` lolos, permintaan akan mencapai `DashboardController::index`.

**Apakah bisa mengubah urutan secara fleksibel?**

**Ya, sangat bisa.**

  * **Secara langsung pada definisi rute:** Anda dapat dengan mudah mengubah urutan dengan menukar posisi alias middleware dalam array `middleware()`:

    ```php
    // Sekarang VerifyEmail berjalan duluan, lalu CheckSubscription
    Route::middleware(['VerifyEmail', 'CheckSubscription'])->group(function () {
        // ...
    });
    ```

  * **Menggunakan `middlewarePriority` di `Kernel.php` (untuk skenario lebih kompleks):**
    Jika Anda memiliki middleware yang harus selalu memiliki prioritas tertentu terlepas dari bagaimana mereka didefinisikan pada rute, atau jika Anda berinteraksi dengan middleware grup, Anda bisa mengatur `$middlewarePriority` di `app/Http/Kernel.php`. Middleware dalam array ini akan selalu dijalankan dalam urutan yang ditentukan, bahkan jika mereka diterapkan dalam urutan yang berbeda pada rute.

    ```php
    // app/Http/Kernel.php
    protected $middlewarePriority = [
        \Illuminate\Session\Middleware\StartSession::class,
        \Illuminate\View\Middleware\ShareErrorsFromSession::class,
        \App\Http\Middleware\Authenticate::class, // Contoh: Pastikan autentikasi selalu di awal
        \App\Http\Middleware\VerifyCsrfToken::class,
        \Illuminate\Routing\Middleware\SubstituteBindings::class,
        \App\Http\Middleware\CheckSubscription::class, // Tentukan prioritas CheckSubscription
        \App\Http\Middleware\VerifyEmail::class,      // Tentukan prioritas VerifyEmail
    ];
    ```

    Middleware yang tidak terdaftar dalam `$middlewarePriority` akan diurutkan secara alfabetis. Mengatur prioritas ini memberikan kontrol yang sangat granular terhadap urutan eksekusi middleware di seluruh aplikasi Anda.

### 7. Jelaskan bagaimana pagination dan filtering bisa digabung dalam satu endpoint API.

Menggabungkan pagination dan filtering dalam satu endpoint API adalah praktik umum untuk menyediakan cara yang fleksibel bagi klien untuk mengambil data yang relevan. Ini biasanya dicapai dengan menggunakan parameter query string di URL.

**Konsep Dasar:**

  * **Pagination:** Mengontrol jumlah item per halaman dan halaman saat ini. Parameter umum: `page`, `per_page` (atau `limit`), `offset`.
  * **Filtering:** Membatasi hasil berdasarkan kriteria tertentu. Parameter umum: `filter[field_name]`, `search`, `status`, `category_id`, `start_date`, `end_date`, dll.
  * **Sorting (opsional tapi umum):** Mengurutkan hasil. Parameter umum: `sort_by`, `sort_order` (atau `order_by`, `direction`).

**Contoh URL Endpoint API:**

```
GET /api/products?page=2&per_page=10&category_id=5&status=published&search=laptop&sort_by=price&sort_order=asc
```

**Implementasi di Laravel (dengan Eloquent):**

Misalkan kita memiliki model `Product` dan ingin membuat endpoint untuk mengambil daftar produk dengan pagination dan filtering.

```php
// app/Http/Controllers/Api/ProductController.php
<?php

namespace App\Http\Controllers\Api;

use App\Http\Controllers\Controller;
use App\Models\Product;
use Illuminate\Http\Request;

class ProductController extends Controller
{
    public function index(Request $request)
    {
        $query = Product::query();

        // 1. Filtering
        if ($request->has('category_id')) {
            $query->where('category_id', $request->input('category_id'));
        }

        if ($request->has('status')) {
            $query->where('status', $request->input('status'));
        }

        if ($request->has('search')) {
            $searchTerm = '%' . $request->input('search') . '%';
            $query->where(function ($q) use ($searchTerm) {
                $q->where('name', 'like', $searchTerm)
                  ->orWhere('description', 'like', $searchTerm);
            });
        }

        // Contoh filtering rentang tanggal
        if ($request->has('start_date') && $request->has('end_date')) {
            $query->whereBetween('created_at', [$request->input('start_date'), $request->input('end_date')]);
        }

        // 2. Sorting (opsional)
        $sortBy = $request->input('sort_by', 'created_at'); // Default sort by created_at
        $sortOrder = $request->input('sort_order', 'desc'); // Default sort order desc

        // Pastikan kolom yang disortir valid untuk menghindari SQL Injection
        $allowedSortColumns = ['id', 'name', 'price', 'created_at'];
        if (in_array($sortBy, $allowedSortColumns)) {
            $query->orderBy($sortBy, $sortOrder);
        }

        // 3. Pagination
        $perPage = $request->input('per_page', 10); // Default 10 item per halaman
        $products = $query->paginate($perPage);

        return response()->json($products);
    }
}
```

**Penjelasan:**

  * **`Product::query()`:** Memulai query builder untuk model `Product`.
  * **Filtering:** Menggunakan `if ($request->has(...))` dan `where()` clause untuk menambahkan kondisi filter ke query berdasarkan parameter yang ada di URL. Anda bisa menambahkan sebanyak mungkin filter yang diperlukan.
  * **Sorting:** Menggunakan `orderBy()` berdasarkan `sort_by` dan `sort_order` dari request. Penting untuk memvalidasi kolom yang diizinkan untuk sorting (`$allowedSortColumns`) untuk mencegah masalah keamanan.
  * **Pagination:** Menggunakan metode `paginate($perPage)` dari Eloquent. Ini secara otomatis menangani `page` parameter dari query string dan mengembalikan objek `LengthAwarePaginator` yang mencakup data item, total item, halaman saat ini, total halaman, dll., yang sangat berguna untuk respon API.

Dengan pendekatan ini, klien API dapat dengan mudah mengirimkan kombinasi parameter untuk mendapatkan data yang spesifik, memfasilitasi aplikasi frontend yang dinamis.

### 8\. Apa perbedaan antara hasManyThrough dan morphMany di Laravel? Berikan contoh studi kasus.

`hasManyThrough` dan `morphMany` adalah dua jenis relasi lanjutan di Eloquent Laravel yang digunakan untuk skenario yang berbeda.

**1. `hasManyThrough`:**

  * **Definisi:** Relasi ini digunakan untuk mengakses model "jauh" melalui model "tengah". Ini berarti Anda memiliki model `A` yang memiliki banyak model `B`, dan model `B` memiliki banyak model `C`. Dengan `hasManyThrough`, Anda dapat langsung mengakses semua `C` yang dimiliki oleh `A`, tanpa perlu memuat `B` secara eksplisit.

  * **Kondisi:** Membutuhkan hubungan yang jelas dan terdefinisi melalui kunci asing.

  * **Studi Kasus:**

      * **Negara -\> Pengguna -\> Postingan:** Sebuah negara memiliki banyak pengguna, dan setiap pengguna memiliki banyak postingan. Anda ingin mendapatkan semua postingan yang berasal dari sebuah negara tertentu.
      * `Country` (id)
      * `User` (id, country\_id)
      * `Post` (id, user\_id)

    **Definisi Relasi (di model `Country`):**

    ```php
    class Country extends Model
    {
        public function posts()
        {
            // Parameter: (Model "jauh", Model "tengah", Kunci asing di model tengah, Kunci asing di model jauh, Kunci lokal model ini, Kunci lokal model tengah)
            return $this->hasManyThrough(Post::class, User::class, 'country_id', 'user_id', 'id', 'id');
        }
    }
    ```

    **Penggunaan:**

    ```php
    $country = Country::find(1);
    $posts = $country->posts; // Mendapatkan semua postingan dari pengguna di negara ini
    ```

**2. `morphMany`:**

  * **Definisi:** Relasi ini digunakan untuk mendefinisikan hubungan polimorfik "satu-ke-banyak". Ini memungkinkan sebuah model memiliki banyak model terkait melalui satu tabel perantara, di mana model terkait dapat dimiliki oleh *berbagai jenis model* lainnya. Tabel perantara ini memiliki kolom `morphable_id` (ID dari model pemilik) dan `morphable_type` (tipe kelas dari model pemilik).

  * **Kondisi:** Digunakan ketika satu model terkait dapat dimiliki oleh lebih dari satu jenis model.

  * **Studi Kasus:**

      * **Komentar/Gambar untuk Postingan, Video, dan Produk:** Anda memiliki model `Comment` atau `Image` yang dapat terkait dengan `Post`, `Video`, atau `Product`.
      * `Comment` (id, body, commentable\_id, commentable\_type)
      * `Post` (id, title)
      * `Video` (id, title)
      * `Product` (id, name)

    **Definisi Relasi (di model `Post`, `Video`, `Product`):**

    ```php
    // Di model Post.php
    class Post extends Model
    {
        public function comments()
        {
            return $this->morphMany(Comment::class, 'commentable'); // 'commentable' adalah nama polimorfik
        }
    }

    // Di model Video.php
    class Video extends Model
    {
        public function comments()
        {
            return $this->morphMany(Comment::class, 'commentable');
        }
    }
    // ... dan di Product.php
    ```

    **Definisi Relasi (di model `Comment`):**

    ```php
    // Di model Comment.php
    class Comment extends Model
    {
        public function commentable()
        {
            return $this->morphTo();
        }
    }
    ```

    **Penggunaan:**

    ```php
    $post = Post::find(1);
    $commentsOnPost = $post->comments; // Mendapatkan komentar untuk postingan ini

    $video = Video::find(5);
    $commentsOnVideo = $video->comments; // Mendapatkan komentar untuk video ini
    ```

**Ringkasan Perbedaan:**

  * **`hasManyThrough`**: Untuk menavigasi dua tingkat relasi "satu-ke-banyak" yang terdefinisi secara statis. Hubungan transitif.
  * **`morphMany`**: Untuk model tunggal yang dapat dimiliki oleh berbagai jenis model lainnya. Hubungan polimorfik.

### 9\. Jelaskan bagaimana kamu akan men-setup Laravel project dari nol di server VPS (tanpa panel seperti cPanel). Sertakan langkah konfigurasi: nginx, PHP, SSL, dan permission.

Menyiapkan proyek Laravel dari nol di VPS melibatkan beberapa langkah penting. Asumsi kita menggunakan Ubuntu Server.

**Prasyarat:**

  * Akses SSH ke VPS Anda (sebagai root atau user dengan `sudo` privileges).
  * Nama domain yang sudah mengarah ke IP VPS Anda.

**Langkah-langkah Konfigurasi:**

**1. Update dan Upgrade Sistem:**

```bash
sudo apt update && sudo apt upgrade -y
```

**2. Instal Nginx:**

```bash
sudo apt install nginx -y
sudo systemctl start nginx
sudo systemctl enable nginx
```

Cek status: `sudo systemctl status nginx`

**3. Instal PHP-FPM dan Ekstensi yang Dibutuhkan:**
Gunakan PHP versi terbaru yang direkomendasikan Laravel (saat ini PHP 8.2 atau lebih tinggi).

```bash
sudo apt install php8.2-fpm php8.2-cli php8.2-mysql php8.2-mbstring php8.2-xml php8.2-bcmath php8.2-zip php8.2-json php8.2-curl php8.2-gd php8.2-tokenizer -y
sudo systemctl start php8.2-fpm
sudo systemctl enable php8.2-fpm
```

Cek status: `sudo systemctl status php8.2-fpm`

**4. Instal Composer:**
Composer adalah package manager untuk PHP, penting untuk Laravel.

```bash
curl -sS https://getcomposer.org/installer | php
sudo mv composer.phar /usr/local/bin/composer
```

**5. Instal Git:**

```bash
sudo apt install git -y
```

**6. Instal Database (MySQL/MariaDB):**

```bash
sudo apt install mysql-server -y
sudo mysql_secure_installation # Ikuti petunjuk untuk mengamankan instalasi MySQL
```

Login ke MySQL dan buat database serta user untuk aplikasi Laravel Anda:

```bash
sudo mysql -u root -p
CREATE DATABASE namadatabaseku CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'usernameku'@'localhost' IDENTIFIED BY 'passwordkuatku';
GRANT ALL PRIVILEGES ON namadatabaseku.* TO 'usernameku'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

**7. Clone Proyek Laravel Anda:**
Pindah ke direktori di mana Anda ingin menyimpan proyek (misalnya `/var/www/`).

```bash
sudo mkdir -p /var/www/your_project_name
cd /var/www/your_project_name
sudo git clone <URL_REPO_ANDA> . # Kloning ke direktori saat ini
```

**8. Konfigurasi Proyek Laravel:**

```bash
cd /var/www/your_project_name
sudo cp .env.example .env
sudo nano .env # Edit file .env:
```

  * `APP_NAME=YourAppName`
  * `APP_ENV=production`
  * `APP_DEBUG=false`
  * `APP_URL=http://yourdomain.com` (akan diubah ke https nanti)
  * Konfigurasi database:
      * `DB_DATABASE=namadatabaseku`
      * `DB_USERNAME=usernameku`
      * `DB_PASSWORD=passwordkuatku`
  * Generate app key:
    ```bash
    php artisan key:generate
    ```
  * Instal dependensi Composer:
    ```bash
    composer install --no-dev --optimize-autoloader
    ```
  * Jalankan migrasi database:
    ```bash
    php artisan migrate --force # --force diperlukan di production
    ```
  * Jalankan seeder (jika ada):
    ```bash
    php artisan db:seed --force
    ```

**9. Konfigurasi Nginx untuk Laravel:**
Buat file konfigurasi Nginx untuk domain Anda.

```bash
sudo nano /etc/nginx/sites-available/your_domain.conf
```

Isi file `your_domain.conf`:

```nginx
server {
    listen 80;
    server_name yourdomain.com www.yourdomain.com;
    root /var/www/your_project_name/public; # Direktori public Laravel

    add_header X-Frame-Options "SAMEORIGIN";
    add_header X-Content-Type-Options "nosniff";

    index index.php index.html index.htm;

    charset utf-8;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location = /favicon.ico { access_log off; log_not_found off; }
    location = /robots.txt  { access_log off; log_not_found off; }

    error_page 404 /index.php;

    location ~ \.php$ {
        fastcgi_pass unix:/var/run/php/php8.2-fpm.sock; # Sesuaikan dengan versi PHP Anda
        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME $realpath_root$fastcgi_script_name;
        include fastcgi_params;
    }

    location ~ /\.(?!well-known).* {
        deny all;
    }
}
```

Aktifkan konfigurasi dengan membuat symlink:

```bash
sudo ln -s /etc/nginx/sites-available/your_domain.conf /etc/nginx/sites-enabled/
```

Uji konfigurasi Nginx dan restart:

```bash
sudo nginx -t
sudo systemctl restart nginx
```

**10. Konfigurasi Permission File dan Direktori:**
Ini sangat krusial untuk keamanan dan agar Laravel dapat menulis log, cache, dll.

```bash
sudo chown -R www-data:www-data /var/www/your_project_name
sudo chmod -R 775 /var/www/your_project_name/storage
sudo chmod -R 775 /var/www/your_project_name/bootstrap/cache
```

`www-data` adalah user/grup default Nginx/PHP-FPM di Ubuntu.

**11. Instalasi SSL dengan Certbot (Let's Encrypt):**

```bash
sudo apt install certbot python3-certbot-nginx -y
sudo certbot --nginx -d yourdomain.com -d www.yourdomain.com
```

Certbot akan secara otomatis mendeteksi konfigurasi Nginx Anda, menginstal sertifikat, dan mengkonfigurasi ulang Nginx untuk HTTPS. Ikuti instruksi pada layar (masukkan email, setuju TOS).

**12. Atur Cron Job Laravel:**
Untuk task scheduling, seperti antrian (queue) atau perintah Artisan terjadwal.

```bash
sudo crontab -e
```

Tambahkan baris berikut di bagian bawah file:

```
* * * * * cd /var/www/your_project_name && php artisan schedule:run >> /dev/null 2>&1
```

**Selesai\!** Proyek Laravel Anda seharusnya sekarang dapat diakses melalui domain Anda dengan HTTPS.

### 10. Buatkan contoh GitHub Actions atau Deployer script untuk deploy Laravel otomatis ke server staging saat ada push ke develop. 

Berikut adalah contoh **GitHub Actions** workflow untuk deploy Laravel otomatis ke server staging saat ada push ke branch `develop`.

**File: `.github/workflows/deploy-staging.yml`**

```yaml
name: Deploy to Staging

on:
  push:
    branches:
      - develop # Akan trigger ketika ada push ke branch 'develop'

jobs:
  deploy:
    runs-on: ubuntu-latest # Runner OS

    steps:
    - name: Checkout Code
      uses: actions/checkout@v4 # Mengambil kode dari repository

    - name: Setup PHP
      uses: shivammathur/setup-php@v2
      with:
        php-version: '8.2' # Sesuaikan dengan versi PHP di server staging Anda
        extensions: mbstring, pdo_mysql, dom, curl, gd, zip # Ekstensi yang dibutuhkan Laravel
        coverage: none # Tidak perlu coverage untuk deploy

    - name: Cache Composer dependencies
      uses: actions/cache@v4
      with:
        path: vendor
        key: ${{ runner.os }}-php-${{ hashFiles('**/composer.lock') }}
        restore-keys: |
          ${{ runner.os }}-php-

    - name: Install Composer Dependencies
      run: composer install --no-dev --prefer-dist --optimize-autoloader

    - name: Generate .env file
      run: |
        echo "APP_NAME=${{ secrets.STAGING_APP_NAME }}" >> .env
        echo "APP_ENV=staging" >> .env
        echo "APP_KEY=${{ secrets.STAGING_APP_KEY }}" >> .env
        echo "APP_URL=${{ secrets.STAGING_APP_URL }}" >> .env
        echo "DB_CONNECTION=${{ secrets.STAGING_DB_CONNECTION }}" >> .env
        echo "DB_HOST=${{ secrets.STAGING_DB_HOST }}" >> .env
        echo "DB_PORT=${{ secrets.STAGING_DB_PORT }}" >> .env
        echo "DB_DATABASE=${{ secrets.STAGING_DB_DATABASE }}" >> .env
        echo "DB_USERNAME=${{ secrets.STAGING_DB_USERNAME }}" >> .env
        echo "DB_PASSWORD=${{ secrets.STAGING_DB_PASSWORD }}" >> .env
        echo "MAIL_MAILER=${{ secrets.STAGING_MAIL_MAILER }}" >> .env
        echo "MAIL_HOST=${{ secrets.STAGING_MAIL_HOST }}" >> .env
        echo "MAIL_PORT=${{ secrets.STAGING_MAIL_PORT }}" >> .env
        echo "MAIL_USERNAME=${{ secrets.STAGING_MAIL_USERNAME }}" >> .env
        echo "MAIL_PASSWORD=${{ secrets.STAGING_MAIL_PASSWORD }}" >> .env
        echo "MAIL_ENCRYPTION=${{ secrets.STAGING_MAIL_ENCRYPTION }}" >> .env
        echo "MAIL_FROM_ADDRESS=${{ secrets.STAGING_MAIL_FROM_ADDRESS }}" >> .env
        echo "MAIL_FROM_NAME=${{ secrets.STAGING_MAIL_FROM_NAME }}" >> .env
        # Tambahkan konfigurasi .env lainnya sesuai kebutuhan
      env:
        # Rahasia-rahasia ini harus disimpan di GitHub Secrets repository Anda
        STAGING_APP_NAME: ${{ secrets.STAGING_APP_NAME }}
        STAGING_APP_KEY: ${{ secrets.STAGING_APP_KEY }}
        STAGING_APP_URL: ${{ secrets.STAGING_APP_URL }}
        STAGING_DB_CONNECTION: ${{ secrets.STAGING_DB_CONNECTION }}
        STAGING_DB_HOST: ${{ secrets.STAGING_DB_HOST }}
        STAGING_DB_PORT: ${{ secrets.STAGING_DB_PORT }}
        STAGING_DB_DATABASE: ${{ secrets.STAGING_DB_DATABASE }}
        STAGING_DB_USERNAME: ${{ secrets.STAGING_DB_USERNAME }}
        STAGING_DB_PASSWORD: ${{ secrets.STAGING_DB_PASSWORD }}
        STAGING_MAIL_MAILER: ${{ secrets.STAGING_MAIL_MAILER }}
        STAGING_MAIL_HOST: ${{ secrets.STAGING_MAIL_HOST }}
        STAGING_MAIL_PORT: ${{ secrets.STAGING_MAIL_PORT }}
        STAGING_MAIL_USERNAME: ${{ secrets.STAGING_MAIL_USERNAME }}
        STAGING_MAIL_PASSWORD: ${{ secrets.STAGING_MAIL_PASSWORD }}
        STAGING_MAIL_ENCRYPTION: ${{ secrets.STAGING_MAIL_ENCRYPTION }}
        STAGING_MAIL_FROM_ADDRESS: ${{ secrets.STAGING_MAIL_FROM_ADDRESS }}
        STAGING_MAIL_FROM_NAME: ${{ secrets.STAGING_MAIL_FROM_NAME }}

    - name: Run Tests (Optional, but recommended for staging)
      run: php artisan test

    - name: Deploy to Staging Server
      uses: appleboy/ssh-action@master
      with:
        host: ${{ secrets.SSH_HOST_STAGING }}
        username: ${{ secrets.SSH_USERNAME_STAGING }}
        key: ${{ secrets.SSH_PRIVATE_KEY_STAGING }}
        script: |
          cd /var/www/your_staging_project_path # Ganti dengan path proyek di server
          git pull origin develop # Pastikan kode terbaru diambil
          composer install --no-dev --optimize-autoloader
          php artisan migrate --force
          php artisan cache:clear
          php artisan config:clear
          php artisan route:clear
          php artisan view:clear
          php artisan storage:link # Jika menggunakan storage symlink
          sudo systemctl reload php8.2-fpm # Restart PHP-FPM jika ada perubahan kode yang butuh reload (misal config)
          # Tambahkan perintah lain yang diperlukan setelah deploy (misal queue restart)
```

**Penjelasan Workflow:**

1.  **`name`**: Nama workflow.
2.  **`on: push: branches: develop`**: Workflow ini akan berjalan setiap kali ada push ke branch `develop`.
3.  **`jobs: deploy`**: Mendefinisikan satu job bernama `deploy`.
4.  **`runs-on: ubuntu-latest`**: Job akan berjalan pada mesin virtual Ubuntu terbaru yang disediakan GitHub Actions.
5.  **`steps`**: Serangkaian langkah yang akan dieksekusi:
      * **`Checkout Code`**: Mengambil kode dari repository GitHub.
      * **`Setup PHP`**: Menginstal versi PHP yang ditentukan beserta ekstensi yang diperlukan.
      * **`Cache Composer dependencies`**: Menggunakan cache untuk dependensi Composer, mempercepat proses instalasi.
      * **`Install Composer Dependencies`**: Menjalankan `composer install` untuk menginstal dependensi tanpa dev dependencies.
      * **`Generate .env file`**: Membuat file `.env` di runner GitHub Actions menggunakan `secrets` yang telah Anda definisikan di repository GitHub (Settings -\> Secrets -\> Actions). Ini penting agar `php artisan` command di langkah berikutnya dapat berjalan dengan konfigurasi yang benar.
      * **`Run Tests` (Opsional)**: Menjalankan unit/feature tests. Ini direkomendasikan untuk memastikan kode tidak merusak fungsionalitas sebelum deploy.
      * **`Deploy to Staging Server`**: Menggunakan `appleboy/ssh-action` untuk terhubung ke server staging via SSH dan menjalankan perintah deployment.
          * `host`, `username`, `key`: Diambil dari GitHub Secrets (Anda harus menambahkan `SSH_HOST_STAGING`, `SSH_USERNAME_STAGING`, dan `SSH_PRIVATE_KEY_STAGING` yang berisi kunci SSH pribadi Anda).
          * `script`: Berisi serangkaian perintah shell yang dijalankan di server staging:
              * `cd /var/www/your_staging_project_path`: Pindah ke direktori proyek Laravel di server.
              * `git pull origin develop`: Mengambil kode terbaru dari branch `develop`.
              * `composer install --no-dev --optimize-autoloader`: Menginstal dependensi Composer.
              * `php artisan migrate --force`: Menjalankan migrasi database.
              * `php artisan cache:clear`, `config:clear`, `route:clear`, `view:clear`: Membersihkan cache Laravel.
              * `php artisan storage:link`: Membuat symlink untuk storage.
              * `sudo systemctl reload php8.2-fpm`: Mereload PHP-FPM untuk memastikan perubahan kode baru di-load.

**Penting:**

  * Ganti `/var/www/your_staging_project_path` dengan path aktual di server Anda.
  * Buat **GitHub Secrets** di repository Anda untuk variabel-variabel sensitif seperti `SSH_HOST_STAGING`, `SSH_USERNAME_STAGING`, `SSH_PRIVATE_KEY_STAGING`, dan semua variabel `.env` untuk staging.
  * Pastikan kunci SSH di server staging Anda telah diatur untuk menerima koneksi dari `SSH_PRIVATE_KEY_STAGING` yang Anda gunakan di GitHub Actions.

  Tentu, saya akan bantu menjawab soal/pertanyaan nomor 11 sampai 20 dari file "Soal Tes Teori Progammer Juli 2025.docx".

### 11\. Jelaskan bagaimana Lazy Loading dan Eager Loading bekerja di Eloquent. Berikan contoh situasi nyata dan dampaknya terhadap performa.

Di Eloquent, **Lazy Loading** dan **Eager Loading** berkaitan dengan cara memuat relasi antar model.

**Lazy Loading (Pemuatan Malas):**

  * **Cara Kerja:** Relasi tidak dimuat sampai Anda secara eksplisit mengaksesnya. Setiap kali Anda mengakses relasi, Eloquent akan menjalankan query database baru untuk mengambil data relasi tersebut.
  * **Contoh:**
    Misalkan Anda memiliki model `User` yang memiliki banyak `Post`.
    ```php
    $users = App\Models\User::all(); // Query: SELECT * FROM users
    foreach ($users as $user) {
        echo $user->name;
        echo $user->posts->count(); // Query: SELECT * FROM posts WHERE user_id = ? (untuk setiap user)
    }
    ```
    Dalam contoh ini, jika ada 1000 user, Eloquent akan menjalankan 1 query untuk mengambil semua user, dan kemudian 1000 query lagi untuk mengambil post dari masing-masing user. Ini dikenal sebagai masalah "N+1 query".
  * **Dampak Terhadap Performa:** Buruk untuk performa jika Anda perlu mengakses relasi untuk banyak model, karena akan menghasilkan banyak query database yang terpisah.

**Eager Loading (Pemuatan Cepat):**

  * **Cara Kerja:** Relasi dimuat bersamaan dengan model utama, biasanya dalam satu atau dua query tambahan (bukan N query terpisah). Ini dilakukan dengan menggunakan metode `with()`.
  * **Contoh:**
    Melanjutkan contoh `User` dan `Post`:
    ```php
    $users = App\Models\User::with('posts')->get(); // Query 1: SELECT * FROM users; Query 2: SELECT * FROM posts WHERE user_id IN (id_user_1, id_user_2, ...)
    foreach ($users as $user) {
        echo $user->name;
        echo $user->posts->count(); // Relasi sudah dimuat, tidak ada query baru
    }
    ```
    Dalam contoh ini, Eloquent hanya akan menjalankan 2 query: satu untuk user dan satu untuk semua post yang terkait dengan user-user tersebut.
  * **Dampak Terhadap Performa:** Sangat meningkatkan performa jika Anda perlu mengakses relasi untuk banyak model, karena mengurangi jumlah total query database secara drastis.

**Situasi Nyata dan Dampak:**

  * **Lazy Loading cocok untuk:** Situasi di mana Anda mungkin tidak selalu membutuhkan data relasi, atau hanya membutuhkan data relasi untuk satu atau beberapa instance model saja. Misalnya, menampilkan detail satu postingan dan mungkin beberapa komentar terkait, bukan daftar semua postingan dengan semua komentar mereka.
  * **Eager Loading cocok untuk:** Situasi di mana Anda pasti akan menggunakan data relasi untuk setiap model dalam sebuah koleksi. Contoh klasik adalah menampilkan daftar artikel di blog dan Anda juga perlu menampilkan nama penulis atau jumlah komentar untuk setiap artikel. Menggunakan eager loading akan mencegah "N+1 problem" dan membuat aplikasi Anda jauh lebih cepat.

### 12\. Laravel menyediakan fitur Job dan Queue. Jelaskan bagaimana kamu akan memproses pengiriman email massal menggunakan fitur ini. Sertakan struktur kode atau pseudocode.

**Job dan Queue** di Laravel digunakan untuk menunda eksekusi tugas yang memakan waktu (seperti pengiriman email, pemrosesan gambar, dll.) ke latar belakang. Ini membuat respons aplikasi tetap cepat bagi pengguna.

**Pengiriman Email Massal Menggunakan Job dan Queue:**

Untuk mengirim email massal, kita tidak ingin proses pengiriman email memblokir permintaan HTTP (misalnya, setelah pengguna mengklik tombol "kirim"). Dengan Job dan Queue, setiap email atau batch email dapat dienkapsulasi dalam sebuah "Job" dan didorong ke "Queue" untuk diproses secara asinkron oleh worker terpisah.

**Langkah-langkah:**

1.  **Buat Mailable:** Representasi email yang akan dikirim.

    ```bash
    php artisan make:mail MassEmail
    ```

    `app/Mail/MassEmail.php`

    ```php
    <?php

    namespace App\Mail;

    use Illuminate\Bus\Queueable;
    use Illuminate\Contracts\Queue\ShouldQueue;
    use Illuminate\Mail\Mailable;
    use Illuminate\Mail\Mailables\Content;
    use Illuminate\Mail\Mailables\Envelope;
    use Illuminate\Queue\SerializesModels;

    class MassEmail extends Mailable
    {
        use Queueable, SerializesModels;

        public $details;

        public function __construct($details)
        {
            $this->details = $details;
        }

        public function envelope(): Envelope
        {
            return new Envelope(
                subject: 'Pemberitahuan Penting: ' . $this->details['subject'],
            );
        }

        public function content(): Content
        {
            return new Content(
                view: 'emails.mass-email', // View Blade untuk konten email
            );
        }
    }
    ```

    `resources/views/emails/mass-email.blade.php` (contoh)

    ```blade
    <h1>{{ $details['subject'] }}</h1>
    <p>{{ $details['body'] }}</p>
    ```

2.  **Buat Job:** Kelas yang akan berisi logika pengiriman email. Job ini akan mengimplementasikan `ShouldQueue`.

    ```bash
    php artisan make:job SendMassEmail
    ```

    `app/Jobs/SendMassEmail.php`

    ```php
    <?php

    namespace App\Jobs;

    use App\Mail\MassEmail;
    use Illuminate\Bus\Queueable;
    use Illuminate\Contracts\Queue\ShouldQueue;
    use Illuminate\Foundation\Bus\Dispatchable;
    use Illuminate\Queue\InteractsWithQueue;
    use Illuminate\Queue\SerializesModels;
    use Illuminate\Support\Facades\Mail;

    class SendMassEmail implements ShouldQueue
    {
        use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

        protected $recipientEmail;
        protected $emailDetails;

        public function __construct(string $recipientEmail, array $emailDetails)
        {
            $this->recipientEmail = $recipientEmail;
            $this->emailDetails = $emailDetails;
        }

        public function handle(): void
        {
            Mail::to($this->recipientEmail)->send(new MassEmail($this->emailDetails));
        }
    }
    ```

3.  **Dispatch Job:** Dari controller atau lokasi lain, Anda mendispatch job ini.

    ```php
    // Misalnya di sebuah Controller
    use App\Jobs\SendMassEmail;
    use App\Models\User; // Misalkan daftar penerima dari model User

    class EmailController extends Controller
    {
        public function sendMassEmails(Request $request)
        {
            $users = User::all(); // Ambil semua user
            $emailDetails = [
                'subject' => $request->input('subject'),
                'body' => $request->input('body'),
            ];

            foreach ($users as $user) {
                // Dispatch job untuk setiap user
                SendMassEmail::dispatch($user->email, $emailDetails);

                // Atau jika ingin menunda pengiriman
                // SendMassEmail::dispatch($user->email, $emailDetails)->delay(now()->addMinutes(5));
            }

            return back()->with('success', 'Email massal sedang dalam antrian pengiriman.');
        }
    }
    ```

4.  **Konfigurasi Queue Driver:** Di `.env`, atur `QUEUE_CONNECTION` (misalnya `redis` atau `database`). Jika `database`, jalankan migrasi `php artisan queue:table` dan `php artisan migrate`.

5.  **Jalankan Queue Worker:** Untuk memproses job yang ada di queue.

    ```bash
    php artisan queue:work
    ```

    Untuk production, gunakan Supervisor atau systemd untuk menjaga worker tetap berjalan.

**Pseudocode (Flow Umum):**

```
FUNGSI kirimEmailMassal(subjek, isiEmail)
    DAFTAR_PENERIMA = ambilSemuaPenggunaDariDatabase()

    UNTUK setiap PENERIMA di DAFTAR_PENERIMA
        BUAT JobKirimEmailBaru(PENERIMA.email, subjek, isiEmail)
        MASUKKAN JobKirimEmailBaru KE Antrian

    KEMBALIKAN "Email sedang diproses di latar belakang."
AKHIR FUNGSI

// Proses di Background Worker
PROSES queue:work
    SEMENTARA AntrianTIDAKKosong
        AMBIL JobBerikutnyaDariAntrian
        EKSEKUSI JobBerikutnya
            // Misalnya, JobKirimEmail:
            // Kirim email ke PENERIMA menggunakan subjek dan isiEmail
            // Catat log jika berhasil/gagal
```

### 13\. Sertakan struktur kode atau pseudocode.

(Sudah disertakan dalam jawaban nomor 12)

### 14\. Bagaimana kamu mengamankan endpoint API Laravel? Jelaskan konsep token-based authentication dan contoh implementasinya dengan Sanctum atau Passport.

Mengamankan endpoint API Laravel sangat penting untuk melindungi data dan memastikan hanya pengguna yang terautentikasi dan terotorisasi yang dapat mengakses sumber daya.

**Konsep Token-Based Authentication:**
Token-based authentication adalah metode autentikasi stateless di mana server tidak menyimpan informasi sesi pengguna. Sebaliknya, setiap permintaan dari klien disertai dengan token (biasanya JWT - JSON Web Token atau opaque token) yang membuktikan identitas klien.

**Flow umum:**

1.  **Login:** Klien mengirimkan kredensial (username/password) ke server.
2.  **Verifikasi & Token Generation:** Server memverifikasi kredensial. Jika valid, server menghasilkan token unik.
3.  **Token Issuance:** Server mengirimkan token kembali ke klien.
4.  **Subsequent Requests:** Klien menyimpan token (misalnya di `localStorage` atau `secure storage`) dan menyertakannya dalam header `Authorization` (biasanya sebagai `Bearer Token`) untuk setiap permintaan berikutnya ke API yang dilindungi.
5.  **Token Validation:** Server menerima permintaan, memvalidasi token (memastikan itu valid, tidak kedaluwarsa, dan tidak diubah), dan jika valid, memproses permintaan. Jika tidak valid, permintaan ditolak.

**Contoh Implementasi dengan Laravel Sanctum:**

Sanctum adalah solusi otentikasi lightweight untuk SPA (Single Page Applications), mobile applications, dan API berbasis token sederhana.

**1. Instalasi:**

```bash
composer require laravel/sanctum
php artisan vendor:publish --provider="Laravel\Sanctum\SanctumServiceProvider"
php artisan migrate
```

**2. Konfigurasi Model User:**
Tambahkan trait `HasApiTokens` ke model `User` Anda.

```php
// app/Models/User.php
<?php

namespace App\Models;

use Illuminate\Contracts\Auth\MustVerifyEmail;
use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Foundation\Auth\User as Authenticatable;
use Illuminate\Notifications\Notifiable;
use Laravel\Sanctum\HasApiTokens; // Import this

class User extends Authenticatable
{
    use HasFactory, Notifiable, HasApiTokens; // Add HasApiTokens
    // ...
}
```

**3. Autentikasi Login (Mendapatkan Token):**
Buat endpoint `/api/login` (atau serupa).

```php
// app/Http/Controllers/Api/AuthController.php
<?php

namespace App\Http\Controllers\Api;

use App\Http\Controllers\Controller;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Auth;
use Illuminate\Support\Facades\Hash;
use App\Models\User;

class AuthController extends Controller
{
    public function login(Request $request)
    {
        $request->validate([
            'email' => 'required|email',
            'password' => 'required',
        ]);

        $user = User::where('email', $request->email)->first();

        if (!$user || !Hash::check($request->password, $user->password)) {
            return response()->json(['message' => 'Kredensial tidak valid'], 401);
        }

        // Buat token. Anda bisa memberi nama token atau menentukan kemampuan (abilities)
        $token = $user->createToken('auth_token')->plainTextToken;

        return response()->json([
            'message' => 'Login berhasil',
            'access_token' => $token,
            'token_type' => 'Bearer',
        ]);
    }

    public function logout(Request $request)
    {
        // Menghapus token yang digunakan saat ini
        $request->user()->currentAccessToken()->delete();

        return response()->json(['message' => 'Logout berhasil']);
    }

    public function user(Request $request)
    {
        return response()->json($request->user());
    }
}
```

**4. Menggunakan Middleware `auth:sanctum`:**
Lindungi endpoint API Anda menggunakan middleware `auth:sanctum`.

```php
// routes/api.php
use App\Http\Controllers\Api\AuthController;
use App\Http\Controllers\Api\PostController; // Contoh controller lain

Route::post('/login', [AuthController::class, 'login']);

Route::middleware('auth:sanctum')->group(function () {
    Route::post('/logout', [AuthController::class, 'logout']);
    Route::get('/user', [AuthController::class, 'user']);

    Route::apiResource('posts', PostController::class);
});
```

**5. Mengirim Permintaan dari Klien:**
Klien harus menyertakan token yang diterima di header `Authorization`.

```
GET /api/user
Host: example.com
Authorization: Bearer YOUR_GENERATED_TOKEN_HERE
```

Dengan Sanctum, setiap kali klien mengirimkan token yang valid di header `Authorization`, Laravel akan secara otomatis mengautentikasi pengguna tersebut.

### 15\. Jika ada dua middleware yang harus dijalankan berurutan (CheckSubscription dan VerifyEmail), bagaimana Laravel mengatur urutannya?

Laravel mengatur urutan middleware berdasarkan dua hal:

1.  **Urutan Registrasi di `app/Http/Kernel.php`:**

      * **`$middleware` array:** Middleware yang tercantum di sini akan berjalan untuk setiap permintaan HTTP secara global, dalam urutan mereka didefinisikan.
      * **`$middlewareGroups` array:** Middleware dalam grup ini (misalnya `web` atau `api`) akan dijalankan dalam urutan yang ditentukan dalam grup tersebut.
      * **`$routeMiddleware` array:** Middleware individual yang diberi alias dan dapat diterapkan ke rute atau grup rute tertentu. Urutan di sini tidak langsung menentukan urutan eksekusi antar alias, melainkan urutan eksekusi *dalam satu alias* jika alias tersebut adalah grup middleware.

2.  **Urutan Penerapan pada Rute:**
    Ketika Anda menerapkan beberapa middleware ke rute (atau grup rute):

    ```php
    Route::middleware(['CheckSubscription', 'VerifyEmail'])->group(function () {
        // Rute-rute yang dilindungi
    });
    ```

    Middleware akan dijalankan dalam urutan yang **didefinisikan dalam array ini**, dari kiri ke kanan. Jadi, `CheckSubscription` akan berjalan terlebih dahulu, diikuti oleh `VerifyEmail`.

**Contoh Kasus `CheckSubscription` dan `VerifyEmail`:**

Jika Anda memiliki `CheckSubscription` dan `VerifyEmail` sebagai alias di `$routeMiddleware`:

```php
// app/Http/Kernel.php
protected $routeMiddleware = [
    // ...
    'CheckSubscription' => \App\Http\Middleware\CheckSubscription::class,
    'VerifyEmail' => \App\Http\Middleware\VerifyEmail::class,
];
```

Dan Anda menerapkannya pada rute seperti ini:

```php
Route::middleware(['CheckSubscription', 'VerifyEmail'])->group(function () {
    Route::get('/dashboard', [DashboardController::class, 'index']);
});
```

Maka:

1.  Permintaan masuk akan melewati `CheckSubscription`.
2.  Jika `CheckSubscription` lolos, permintaan akan diteruskan ke `VerifyEmail`.
3.  Jika `VerifyEmail` lolos, permintaan akan mencapai `DashboardController::index`.

### 16\. Apakah bisa mengubah urutan secara fleksibel?

**Ya, sangat bisa.**

  * **Secara langsung pada definisi rute:** Anda dapat dengan mudah mengubah urutan dengan menukar posisi alias middleware dalam array `middleware()`:

    ```php
    // Sekarang VerifyEmail berjalan duluan, lalu CheckSubscription
    Route::middleware(['VerifyEmail', 'CheckSubscription'])->group(function () {
        // ...
    });
    ```

  * **Menggunakan `middlewarePriority` di `Kernel.php` (untuk skenario lebih kompleks):**
    Jika Anda memiliki middleware yang harus selalu memiliki prioritas tertentu terlepas dari bagaimana mereka didefinisikan pada rute, atau jika Anda berinteraksi dengan middleware grup, Anda bisa mengatur `$middlewarePriority` di `app/Http/Kernel.php`. Middleware dalam array ini akan selalu dijalankan dalam urutan yang ditentukan, bahkan jika mereka diterapkan dalam urutan yang berbeda pada rute.

    ```php
    // app/Http/Kernel.php
    protected $middlewarePriority = [
        \Illuminate\Session\Middleware\StartSession::class,
        \Illuminate\View\Middleware\ShareErrorsFromSession::class,
        \App\Http\Middleware\Authenticate::class, // Contoh: Pastikan autentikasi selalu di awal
        \App\Http\Middleware\VerifyCsrfToken::class,
        \Illuminate\Routing\Middleware\SubstituteBindings::class,
        \App\Http\Middleware\CheckSubscription::class, // Tentukan prioritas CheckSubscription
        \App\Http\Middleware\VerifyEmail::class,      // Tentukan prioritas VerifyEmail
    ];
    ```

    Middleware yang tidak terdaftar dalam `$middlewarePriority` akan diurutkan secara alfabetis. Mengatur prioritas ini memberikan kontrol yang sangat granular terhadap urutan eksekusi middleware di seluruh aplikasi Anda.

### 17\. Jelaskan bagaimana pagination dan filtering bisa digabung dalam satu endpoint API.

Menggabungkan pagination dan filtering dalam satu endpoint API adalah praktik umum untuk menyediakan cara yang fleksibel bagi klien untuk mengambil data yang relevan. Ini biasanya dicapai dengan menggunakan parameter query string di URL.

**Konsep Dasar:**

  * **Pagination:** Mengontrol jumlah item per halaman dan halaman saat ini. Parameter umum: `page`, `per_page` (atau `limit`), `offset`.
  * **Filtering:** Membatasi hasil berdasarkan kriteria tertentu. Parameter umum: `filter[field_name]`, `search`, `status`, `category_id`, `start_date`, `end_date`, dll.
  * **Sorting (opsional tapi umum):** Mengurutkan hasil. Parameter umum: `sort_by`, `sort_order` (atau `order_by`, `direction`).

**Contoh URL Endpoint API:**

```
GET /api/products?page=2&per_page=10&category_id=5&status=published&search=laptop&sort_by=price&sort_order=asc
```

**Implementasi di Laravel (dengan Eloquent):**

Misalkan kita memiliki model `Product` dan ingin membuat endpoint untuk mengambil daftar produk dengan pagination dan filtering.

```php
// app/Http/Controllers/Api/ProductController.php
<?php

namespace App\Http\Controllers\Api;

use App\Http\Controllers\Controller;
use App\Models\Product;
use Illuminate\Http\Request;

class ProductController extends Controller
{
    public function index(Request $request)
    {
        $query = Product::query();

        // 1. Filtering
        if ($request->has('category_id')) {
            $query->where('category_id', $request->input('category_id'));
        }

        if ($request->has('status')) {
            $query->where('status', $request->input('status'));
        }

        if ($request->has('search')) {
            $searchTerm = '%' . $request->input('search') . '%';
            $query->where(function ($q) use ($searchTerm) {
                $q->where('name', 'like', $searchTerm)
                  ->orWhere('description', 'like', $searchTerm);
            });
        }

        // Contoh filtering rentang tanggal
        if ($request->has('start_date') && $request->has('end_date')) {
            $query->whereBetween('created_at', [$request->input('start_date'), $request->input('end_date')]);
        }

        // 2. Sorting (opsional)
        $sortBy = $request->input('sort_by', 'created_at'); // Default sort by created_at
        $sortOrder = $request->input('sort_order', 'desc'); // Default sort order desc

        // Pastikan kolom yang disortir valid untuk menghindari SQL Injection
        $allowedSortColumns = ['id', 'name', 'price', 'created_at'];
        if (in_array($sortBy, $allowedSortColumns)) {
            $query->orderBy($sortBy, $sortOrder);
        }

        // 3. Pagination
        $perPage = $request->input('per_page', 10); // Default 10 item per halaman
        $products = $query->paginate($perPage);

        return response()->json($products);
    }
}
```

**Penjelasan:**

  * **`Product::query()`:** Memulai query builder untuk model `Product`.
  * **Filtering:** Menggunakan `if ($request->has(...))` dan `where()` clause untuk menambahkan kondisi filter ke query berdasarkan parameter yang ada di URL. Anda bisa menambahkan sebanyak mungkin filter yang diperlukan.
  * **Sorting:** Menggunakan `orderBy()` berdasarkan `sort_by` dan `sort_order` dari request. Penting untuk memvalidasi kolom yang diizinkan untuk sorting (`$allowedSortColumns`) untuk mencegah masalah keamanan.
  * **Pagination:** Menggunakan metode `paginate($perPage)` dari Eloquent. Ini secara otomatis menangani `page` parameter dari query string dan mengembalikan objek `LengthAwarePaginator` yang mencakup data item, total item, halaman saat ini, total halaman, dll., yang sangat berguna untuk respon API.

Dengan pendekatan ini, klien API dapat dengan mudah mengirimkan kombinasi parameter untuk mendapatkan data yang spesifik, memfasilitasi aplikasi frontend yang dinamis.

### 18\. Apa perbedaan antara hasManyThrough dan morphMany di Laravel? Berikan contoh studi kasus.

`hasManyThrough` dan `morphMany` adalah dua jenis relasi lanjutan di Eloquent Laravel yang digunakan untuk skenario yang berbeda.

**1. `hasManyThrough`:**

  * **Definisi:** Relasi ini digunakan untuk mengakses model "jauh" melalui model "tengah". Ini berarti Anda memiliki model `A` yang memiliki banyak model `B`, dan model `B` memiliki banyak model `C`. Dengan `hasManyThrough`, Anda dapat langsung mengakses semua `C` yang dimiliki oleh `A`, tanpa perlu memuat `B` secara eksplisit.

  * **Kondisi:** Membutuhkan hubungan yang jelas dan terdefinisi melalui kunci asing.

  * **Studi Kasus:**

      * **Negara -\> Pengguna -\> Postingan:** Sebuah negara memiliki banyak pengguna, dan setiap pengguna memiliki banyak postingan. Anda ingin mendapatkan semua postingan yang berasal dari sebuah negara tertentu.
      * `Country` (id)
      * `User` (id, country\_id)
      * `Post` (id, user\_id)

    **Definisi Relasi (di model `Country`):**

    ```php
    class Country extends Model
    {
        public function posts()
        {
            // Parameter: (Model "jauh", Model "tengah", Kunci asing di model tengah, Kunci asing di model jauh, Kunci lokal model ini, Kunci lokal model tengah)
            return $this->hasManyThrough(Post::class, User::class, 'country_id', 'user_id', 'id', 'id');
        }
    }
    ```

    **Penggunaan:**

    ```php
    $country = Country::find(1);
    $posts = $country->posts; // Mendapatkan semua postingan dari pengguna di negara ini
    ```

**2. `morphMany`:**

  * **Definisi:** Relasi ini digunakan untuk mendefinisikan hubungan polimorfik "satu-ke-banyak". Ini memungkinkan sebuah model memiliki banyak model terkait melalui satu tabel perantara, di mana model terkait dapat dimiliki oleh *berbagai jenis model* lainnya. Tabel perantara ini memiliki kolom `morphable_id` (ID dari model pemilik) dan `morphable_type` (tipe kelas dari model pemilik).

  * **Kondisi:** Digunakan ketika satu model terkait dapat dimiliki oleh lebih dari satu jenis model.

  * **Studi Kasus:**

      * **Komentar/Gambar untuk Postingan, Video, dan Produk:** Anda memiliki model `Comment` atau `Image` yang dapat terkait dengan `Post`, `Video`, atau `Product`.
      * `Comment` (id, body, commentable\_id, commentable\_type)
      * `Post` (id, title)
      * `Video` (id, title)
      * `Product` (id, name)

    **Definisi Relasi (di model `Post`, `Video`, `Product`):**

    ```php
    // Di model Post.php
    class Post extends Model
    {
        public function comments()
        {
            return $this->morphMany(Comment::class, 'commentable'); // 'commentable' adalah nama polimorfik
        }
    }

    // Di model Video.php
    class Video extends Model
    {
        public function comments()
        {
            return $this->morphMany(Comment::class, 'commentable');
        }
    }
    // ... dan di Product.php
    ```

    **Definisi Relasi (di model `Comment`):**

    ```php
    // Di model Comment.php
    class Comment extends Model
    {
        public function commentable()
        {
            return $this->morphTo();
        }
    }
    ```

    **Penggunaan:**

    ```php
    $post = Post::find(1);
    $commentsOnPost = $post->comments; // Mendapatkan komentar untuk postingan ini

    $video = Video::find(5);
    $commentsOnVideo = $video->comments; // Mendapatkan komentar untuk video ini
    ```

**Ringkasan Perbedaan:**

  * **`hasManyThrough`**: Untuk menavigasi dua tingkat relasi "satu-ke-banyak" yang terdefinisi secara statis. Hubungan transitif.
  * **`morphMany`**: Untuk model tunggal yang dapat dimiliki oleh berbagai jenis model lainnya. Hubungan polimorfik.

### 19\. Jelaskan bagaimana kamu akan men-setup Laravel project dari nol di server VPS (tanpa panel seperti cPanel). Sertakan langkah konfigurasi: nginx, PHP, SSL, dan permission.

Menyiapkan proyek Laravel dari nol di VPS melibatkan beberapa langkah penting. Asumsi kita menggunakan Ubuntu Server.

**Prasyarat:**

  * Akses SSH ke VPS Anda (sebagai root atau user dengan `sudo` privileges).
  * Nama domain yang sudah mengarah ke IP VPS Anda.

**Langkah-langkah Konfigurasi:**

**1. Update dan Upgrade Sistem:**

```bash
sudo apt update && sudo apt upgrade -y
```

**2. Instal Nginx:**

```bash
sudo apt install nginx -y
sudo systemctl start nginx
sudo systemctl enable nginx
```

Cek status: `sudo systemctl status nginx`

**3. Instal PHP-FPM dan Ekstensi yang Dibutuhkan:**
Gunakan PHP versi terbaru yang direkomendasikan Laravel (saat ini PHP 8.2 atau lebih tinggi).

```bash
sudo apt install php8.2-fpm php8.2-cli php8.2-mysql php8.2-mbstring php8.2-xml php8.2-bcmath php8.2-zip php8.2-json php8.2-curl php8.2-gd php8.2-tokenizer -y
sudo systemctl start php8.2-fpm
sudo systemctl enable php8.2-fpm
```

Cek status: `sudo systemctl status php8.2-fpm`

**4. Instal Composer:**
Composer adalah package manager untuk PHP, penting untuk Laravel.

```bash
curl -sS https://getcomposer.org/installer | php
sudo mv composer.phar /usr/local/bin/composer
```

**5. Instal Git:**

```bash
sudo apt install git -y
```

**6. Instal Database (MySQL/MariaDB):**

```bash
sudo apt install mysql-server -y
sudo mysql_secure_installation # Ikuti petunjuk untuk mengamankan instalasi MySQL
```

Login ke MySQL dan buat database serta user untuk aplikasi Laravel Anda:

```bash
sudo mysql -u root -p
CREATE DATABASE namadatabaseku CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'usernameku'@'localhost' IDENTIFIED BY 'passwordkuatku';
GRANT ALL PRIVILEGES ON namadatabaseku.* TO 'usernameku'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

**7. Clone Proyek Laravel Anda:**
Pindah ke direktori di mana Anda ingin menyimpan proyek (misalnya `/var/www/`).

```bash
sudo mkdir -p /var/www/your_project_name
cd /var/www/your_project_name
sudo git clone <URL_REPO_ANDA> . # Kloning ke direktori saat ini
```

**8. Konfigurasi Proyek Laravel:**

```bash
cd /var/www/your_project_name
sudo cp .env.example .env
sudo nano .env # Edit file .env:
```

  * `APP_NAME=YourAppName`
  * `APP_ENV=production`
  * `APP_DEBUG=false`
  * `APP_URL=http://yourdomain.com` (akan diubah ke https nanti)
  * Konfigurasi database:
      * `DB_DATABASE=namadatabaseku`
      * `DB_USERNAME=usernameku`
      * `DB_PASSWORD=passwordkuatku`
  * Generate app key:
    ```bash
    php artisan key:generate
    ```
  * Instal dependensi Composer:
    ```bash
    composer install --no-dev --optimize-autoloader
    ```
  * Jalankan migrasi database:
    ```bash
    php artisan migrate --force # --force diperlukan di production
    ```
  * Jalankan seeder (jika ada):
    ```bash
    php artisan db:seed --force
    ```

**9. Konfigurasi Nginx untuk Laravel:**
Buat file konfigurasi Nginx untuk domain Anda.

```bash
sudo nano /etc/nginx/sites-available/your_domain.conf
```

Isi file `your_domain.conf`:

```nginx
server {
    listen 80;
    server_name yourdomain.com www.yourdomain.com;
    root /var/www/your_project_name/public; # Direktori public Laravel

    add_header X-Frame-Options "SAMEORIGIN";
    add_header X-Content-Type-Options "nosniff";

    index index.php index.html index.htm;

    charset utf-8;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location = /favicon.ico { access_log off; log_not_found off; }
    location = /robots.txt  { access_log off; log_not_found off; }

    error_page 404 /index.php;

    location ~ \.php$ {
        fastcgi_pass unix:/var/run/php/php8.2-fpm.sock; # Sesuaikan dengan versi PHP Anda
        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME $realpath_root$fastcgi_script_name;
        include fastcgi_params;
    }

    location ~ /\.(?!well-known).* {
        deny all;
    }
}
```

Aktifkan konfigurasi dengan membuat symlink:

```bash
sudo ln -s /etc/nginx/sites-available/your_domain.conf /etc/nginx/sites-enabled/
```

Uji konfigurasi Nginx dan restart:

```bash
sudo nginx -t
sudo systemctl restart nginx
```

**10. Konfigurasi Permission File dan Direktori:**
Ini sangat krusial untuk keamanan dan agar Laravel dapat menulis log, cache, dll. 

```bash
sudo chown -R www-data:www-data /var/www/your_project_name
sudo chmod -R 775 /var/www/your_project_name/storage
sudo chmod -R 775 /var/www/your_project_name/bootstrap/cache
```

`www-data` adalah user/grup default Nginx/PHP-FPM di Ubuntu.

**11. Instalasi SSL dengan Certbot (Let's Encrypt):**

```bash
sudo apt install certbot python3-certbot-nginx -y
sudo certbot --nginx -d yourdomain.com -d www.yourdomain.com
```

Certbot akan secara otomatis mendeteksi konfigurasi Nginx Anda, menginstal sertifikat, dan mengkonfigurasi ulang Nginx untuk HTTPS. Ikuti instruksi pada layar (masukkan email, setuju TOS).

**12. Atur Cron Job Laravel:**
Untuk task scheduling, seperti antrian (queue) atau perintah Artisan terjadwal.

```bash
sudo crontab -e
```

Tambahkan baris berikut di bagian bawah file:

```
* * * * * cd /var/www/your_project_name && php artisan schedule:run >> /dev/null 2>&1
```

**Selesai\!** Proyek Laravel Anda seharusnya sekarang dapat diakses melalui domain Anda dengan HTTPS.

### 20\. Buatkan contoh GitHub Actions atau Deployer script untuk deploy Laravel otomatis ke server staging saat ada push ke develop.

Berikut adalah contoh **GitHub Actions** workflow untuk deploy Laravel otomatis ke server staging saat ada push ke branch `develop`. 

**File: `.github/workflows/deploy-staging.yml`**

```yaml
name: Deploy to Staging

on:
  push:
    branches:
      - develop # Akan trigger ketika ada push ke branch 'develop'

jobs:
  deploy:
    runs-on: ubuntu-latest # Runner OS

    steps:
    - name: Checkout Code
      uses: actions/checkout@v4 # Mengambil kode dari repository

    - name: Setup PHP
      uses: shivammathur/setup-php@v2
      with:
        php-version: '8.2' # Sesuaikan dengan versi PHP di server staging Anda
        extensions: mbstring, pdo_mysql, dom, curl, gd, zip # Ekstensi yang dibutuhkan Laravel
        coverage: none # Tidak perlu coverage untuk deploy

    - name: Cache Composer dependencies
      uses: actions/cache@v4
      with:
        path: vendor
        key: ${{ runner.os }}-php-${{ hashFiles('**/composer.lock') }}
        restore-keys: |
          ${{ runner.os }}-php-

    - name: Install Composer Dependencies
      run: composer install --no-dev --prefer-dist --optimize-autoloader

    - name: Generate .env file
      run: |
        echo "APP_NAME=${{ secrets.STAGING_APP_NAME }}" >> .env
        echo "APP_ENV=staging" >> .env
        echo "APP_KEY=${{ secrets.STAGING_APP_KEY }}" >> .env
        echo "APP_URL=${{ secrets.STAGING_APP_URL }}" >> .env
        echo "DB_CONNECTION=${{ secrets.STAGING_DB_CONNECTION }}" >> .env
        echo "DB_HOST=${{ secrets.STAGING_DB_HOST }}" >> .env
        echo "DB_PORT=${{ secrets.STAGING_DB_PORT }}" >> .env
        echo "DB_DATABASE=${{ secrets.STAGING_DB_DATABASE }}" >> .env
        echo "DB_USERNAME=${{ secrets.STAGING_DB_USERNAME }}" >> .env
        echo "DB_PASSWORD=${{ secrets.STAGING_DB_PASSWORD }}" >> .env
        echo "MAIL_MAILER=${{ secrets.STAGING_MAIL_MAILER }}" >> .env
        echo "MAIL_HOST=${{ secrets.STAGING_MAIL_HOST }}" >> .env
        echo "MAIL_PORT=${{ secrets.STAGING_MAIL_PORT }}" >> .env
        echo "MAIL_USERNAME=${{ secrets.STAGING_MAIL_USERNAME }}" >> .env
        echo "MAIL_PASSWORD=${{ secrets.STAGING_MAIL_PASSWORD }}" >> .env
        echo "MAIL_ENCRYPTION=${{ secrets.STAGING_MAIL_ENCRYPTION }}" >> .env
        echo "MAIL_FROM_ADDRESS=${{ secrets.STAGING_MAIL_FROM_ADDRESS }}" >> .env
        echo "MAIL_FROM_NAME=${{ secrets.STAGING_MAIL_FROM_NAME }}" >> .env
        # Tambahkan konfigurasi .env lainnya sesuai kebutuhan
      env:
        # Rahasia-rahasia ini harus disimpan di GitHub Secrets repository Anda
        STAGING_APP_NAME: ${{ secrets.STAGING_APP_NAME }}
        STAGING_APP_KEY: ${{ secrets.STAGING_APP_KEY }}
        STAGING_APP_URL: ${{ secrets.STAGING_APP_URL }}
        STAGING_DB_CONNECTION: ${{ secrets.STAGING_DB_CONNECTION }}
        STAGING_DB_HOST: ${{ secrets.STAGING_DB_HOST }}
        STAGING_DB_PORT: ${{ secrets.STAGING_DB_PORT }}
        STAGING_DB_DATABASE: ${{ secrets.STAGING_DB_DATABASE }}
        STAGING_DB_USERNAME: ${{ secrets.STAGING_DB_USERNAME }}
        STAGING_DB_PASSWORD: ${{ secrets.STAGING_DB_PASSWORD }}
        STAGING_MAIL_MAILER: ${{ secrets.STAGING_MAIL_MAILER }}
        STAGING_MAIL_HOST: ${{ secrets.STAGING_MAIL_HOST }}
        STAGING_MAIL_PORT: ${{ secrets.STAGING_MAIL_PORT }}
        STAGING_MAIL_USERNAME: ${{ secrets.STAGING_MAIL_USERNAME }}
        STAGING_MAIL_PASSWORD: ${{ secrets.STAGING_MAIL_PASSWORD }}
        STAGING_MAIL_ENCRYPTION: ${{ secrets.STAGING_MAIL_ENCRYPTION }}
        STAGING_MAIL_FROM_ADDRESS: ${{ secrets.STAGING_MAIL_FROM_ADDRESS }}
        STAGING_MAIL_FROM_NAME: ${{ secrets.STAGING_MAIL_FROM_NAME }}

    - name: Run Tests (Optional, but recommended for staging)
      run: php artisan test

    - name: Deploy to Staging Server
      uses: appleboy/ssh-action@master
      with:
        host: ${{ secrets.SSH_HOST_STAGING }}
        username: ${{ secrets.SSH_USERNAME_STAGING }}
        key: ${{ secrets.SSH_PRIVATE_KEY_STAGING }}
        script: |
          cd /var/www/your_staging_project_path # Ganti dengan path proyek di server
          git pull origin develop # Pastikan kode terbaru diambil
          composer install --no-dev --optimize-autoloader
          php artisan migrate --force
          php artisan cache:clear
          php artisan config:clear
          php artisan route:clear
          php artisan view:clear
          php artisan storage:link # Jika menggunakan storage symlink
          sudo systemctl reload php8.2-fpm # Restart PHP-FPM jika ada perubahan kode yang butuh reload (misal config)
          # Tambahkan perintah lain yang diperlukan setelah deploy (misal queue restart)
```

**Penjelasan Workflow:** 

1.  **`name`**: Nama workflow.
2.  **`on: push: branches: develop`**: Workflow ini akan berjalan setiap kali ada push ke branch `develop`.
3.  **`jobs: deploy`**: Mendefinisikan satu job bernama `deploy`.
4.  **`runs-on: ubuntu-latest`**: Job akan berjalan pada mesin virtual Ubuntu terbaru yang disediakan GitHub Actions.
5.  **`steps`**: Serangkaian langkah yang akan dieksekusi:
      * **`Checkout Code`**: Mengambil kode dari repository GitHub.
      * **`Setup PHP`**: Menginstal versi PHP yang ditentukan beserta ekstensi yang diperlukan.
      * **`Cache Composer dependencies`**: Menggunakan cache untuk dependensi Composer, mempercepat proses instalasi.
      * **`Install Composer Dependencies`**: Menjalankan `composer install` untuk menginstal dependensi tanpa dev dependencies.
      * **`Generate .env file`**: Membuat file `.env` di runner GitHub Actions menggunakan `secrets` yang telah Anda definisikan di repository GitHub (Settings -\> Secrets -\> Actions). Ini penting agar `php artisan` command di langkah berikutnya dapat berjalan dengan konfigurasi yang benar.
      * **`Run Tests` (Opsional)**: Menjalankan unit/feature tests. Ini direkomendasikan untuk memastikan kode tidak merusak fungsionalitas sebelum deploy.
      * **`Deploy to Staging Server`**: Menggunakan `appleboy/ssh-action` untuk terhubung ke server staging via SSH dan menjalankan perintah deployment.
          * `host`, `username`, `key`: Diambil dari GitHub Secrets (Anda harus menambahkan `SSH_HOST_STAGING`, `SSH_USERNAME_STAGING`, dan `SSH_PRIVATE_KEY_STAGING` yang berisi kunci SSH pribadi Anda).
          * `script`: Berisi serangkaian perintah shell yang dijalankan di server staging:
              * `cd /var/www/your_staging_project_path`: Pindah ke direktori proyek Laravel di server.
              * `git pull origin develop`: Mengambil kode terbaru dari branch `develop`.
              * `composer install --no-dev --optimize-autoloader`: Menginstal dependensi Composer.
              * `php artisan migrate --force`: Menjalankan migrasi database.
              * `php artisan cache:clear`, `config:clear`, `route:clear`, `view:clear`: Membersihkan cache Laravel.
              * `php artisan storage:link`: Membuat symlink untuk storage.
              * `sudo systemctl reload php8.2-fpm`: Mereload PHP-FPM untuk memastikan perubahan kode baru di-load.

**Penting:** 

  * Ganti `/var/www/your_staging_project_path` dengan path aktual di server Anda.
  * Buat **GitHub Secrets** di repository Anda untuk variabel-variabel sensitif seperti `SSH_HOST_STAGING`, `SSH_USERNAME_STAGING`, `SSH_PRIVATE_KEY_STAGING`, dan semua variabel `.env` untuk staging.
  * Pastikan kunci SSH di server staging Anda telah diatur untuk menerima koneksi dari `SSH_PRIVATE_KEY_STAGING` yang Anda gunakan di GitHub Actions.

Tentu, saya akan bantu menjawab soal/pertanyaan nomor 21 sampai 30 dari file "Soal Tes Teori Progammer Juli 2025.docx".

### 21\. Jelaskan perbedaan antara Session dan Cache di Laravel. Kapan kamu akan menggunakan masing-masing?

**Session (Sesi):**

  * **Definisi:** Session adalah mekanisme untuk menyimpan data pengguna di antara permintaan HTTP. Data sesi biasanya disimpan di server (default: file, database, Redis, Memcached) dan diidentifikasi oleh cookie sesi yang dikirim ke browser pengguna. Setiap pengguna memiliki sesi uniknya sendiri.
  * **Tujuan:** Untuk melacak status pengguna dan menyimpan data spesifik pengguna selama kunjungan mereka ke aplikasi.
  * **Contoh Penggunaan:**
      * Menyimpan ID pengguna setelah login.
      * Menyimpan pesan flash (misalnya, "Data berhasil disimpan\!" yang hanya muncul sekali).
      * Keranjang belanja (cart) pengguna yang belum checkout.
      * Preferensi pengguna (misalnya, tema gelap/terang).
  * **Umur Data:** Sesuai umur sesi (biasanya dikonfigurasi, misalnya 2 jam), atau sampai pengguna logout/menutup browser (tergantung konfigurasi cookie). Data sesi bersifat individual untuk setiap pengguna.
  * **Laravel Implementation:** Menggunakan facade `Session` atau helper `session()`.
    ```php
    session(['key' => 'value']);
    $value = session('key');
    session()->flash('success', 'Operation successful!');
    ```

**Cache (Tembolok):**

  * **Definisi:** Cache adalah mekanisme untuk menyimpan data yang sering diakses dan mahal untuk dihasilkan kembali (misalnya, hasil query database yang kompleks, data konfigurasi, hasil render view) untuk sementara waktu di lokasi yang lebih cepat (memori, file, Redis, Memcached). Tujuannya adalah untuk mempercepat waktu respons aplikasi dengan menghindari perhitungan atau pengambilan data yang sama berulang kali.
  * **Tujuan:** Untuk meningkatkan performa aplikasi secara keseluruhan dengan mengurangi beban pada database atau CPU.
  * **Contoh Penggunaan:**
      * Daftar produk populer.
      * Hasil query yang jarang berubah (misalnya, daftar kategori, setting aplikasi).
      * Data yang digunakan oleh banyak pengguna secara bersamaan.
      * Hasil render Blade view (cache view).
  * **Umur Data:** Dapat dikonfigurasi untuk kedaluwarsa setelah jangka waktu tertentu, atau sampai dihapus secara manual. Data cache bersifat global (dapat diakses oleh semua pengguna) atau dapat di-scoped berdasarkan kunci tertentu.
  * **Laravel Implementation:** Menggunakan facade `Cache` atau helper `cache()`.
    ```php
    Cache::put('popular_products', $products, $minutes = 60);
    $products = Cache::get('popular_products');
    Cache::forget('popular_products');
    ```

**Perbedaan Utama:**
| Fitur           | Session                                | Cache                                     |
| :-------------- | :------------------------------------- | :---------------------------------------- |
| **Lingkup** | Spesifik pengguna (per permintaan)     | Umum (untuk semua pengguna) atau tertentu |
| **Tujuan** | Melacak status pengguna, data temporal | Meningkatkan performa aplikasi              |
| **Contoh Data** | Login user ID, flash messages          | Data produk, konfigurasi, hasil query     |
| **Umur** | Terkait dengan durasi sesi pengguna    | Terkonfigurasi (kedaluwarsa atau manual)  |
| **Penyimpanan** | File, database, Redis, Memcached       | File, Redis, Memcached, database          |

**Kapan Menggunakan Masing-masing:**

  * **Gunakan Session:** Ketika Anda perlu menyimpan informasi yang hanya relevan untuk satu pengguna tertentu dan selama durasi kunjungan mereka.
  * **Gunakan Cache:** Ketika Anda perlu menyimpan data yang sama dan sering diakses oleh banyak pengguna atau data yang mahal untuk dihasilkan, untuk mengurangi beban pada sumber daya backend dan mempercepat aplikasi.

### 22\. Jelaskan konsep Service Pattern dan bagaimana perbedaannya dengan Repository Pattern. Berikan contoh kapan menggunakan Service Pattern.

**Service Pattern (Pola Layanan):**

  * **Definisi:** Service Pattern adalah pola desain yang digunakan untuk mengisolasi logika bisnis kompleks yang melibatkan interaksi dengan beberapa model atau komponen lain. Sebuah "Service" adalah kelas yang mengkoordinasikan operasi di berbagai bagian aplikasi, seringkali melibatkan beberapa repository, notifikasi, event, atau logika validasi yang kompleks.
  * **Tujuan:**
      * **Merapikan Controller:** Menjaga controller tetap ramping (`thin controllers`) dengan memindahkan logika bisnis yang kompleks keluar dari mereka.
      * **Reusabilitas:** Logika bisnis yang sama dapat digunakan kembali di berbagai tempat (controller, job, command, event listener).
      * **Keterbacaan dan Kemudahan Uji:** Kode bisnis lebih terorganisir, mudah dibaca, dan diuji secara terpisah.
  * **Kapan Menggunakan Service Pattern:**
    Ketika Anda memiliki operasi yang:
      * Melibatkan banyak langkah atau koordinasi antara beberapa model/repository.
      * Memiliki logika bisnis yang kompleks.
      * Memicu beberapa efek samping (misalnya, mengirim email, membuat entri log, memicu event).
      * Mungkin perlu digunakan dari berbagai titik masuk aplikasi (misalnya, API, web UI, console command).

**Contoh Kasus Penggunaan Service Pattern:**
Misalkan Anda memiliki operasi pendaftaran pengguna yang melibatkan:

1.  Membuat entri pengguna di database.
2.  Mengirim email verifikasi.
3.  Membuat entri profil default.
4.  Memicu event `UserRegistered`.

Tanpa Service Pattern, semua logika ini mungkin akan berakhir di `AuthController@register`. Dengan Service Pattern, Anda dapat membuat `UserService`:

```php
// app/Services/UserService.php
<?php

namespace App\Services;

use App\Models\User;
use App\Repositories\UserRepository; // Menggunakan Repository Pattern
use App\Mail\UserRegisteredMail;
use Illuminate\Support\Facades\Mail;
use Illuminate\Support\Facades\Hash;
use Illuminate\Support\Facades\Event;
use App\Events\UserRegistered; // Contoh event

class UserService
{
    protected $userRepository;

    public function __construct(UserRepository $userRepository)
    {
        $this->userRepository = $userRepository;
    }

    public function registerUser(array $data): User
    {
        // 1. Validasi (bisa juga dilakukan di Form Request)
        // ...

        // 2. Buat user
        $data['password'] = Hash::make($data['password']);
        $user = $this->userRepository->create($data); // Menggunakan Repository

        // 3. Kirim email verifikasi
        Mail::to($user->email)->send(new UserRegisteredMail($user));

        // 4. Buat profil default (jika ada model Profile)
        // $user->profile()->create(['bio' => 'Welcome new user!']);

        // 5. Memicu event
        Event::dispatch(new UserRegistered($user));

        return $user;
    }

    // Metode lain seperti updateProfile, deactivateUser, dll.
}
```

**Penggunaan di Controller:**

```php
// app/Http/Controllers/AuthController.php
<?php

namespace App\Http\Controllers;

use App\Services\UserService;
use Illuminate\Http\Request;
use App\Http\Requests\RegisterUserRequest; // Menggunakan Form Request untuk validasi

class AuthController extends Controller
{
    protected $userService;

    public function __construct(UserService $userService)
    {
        $this->userService = $userService;
    }

    public function register(RegisterUserRequest $request)
    {
        try {
            $user = $this->userService->registerUser($request->validated());
            return response()->json(['message' => 'User registered successfully', 'user' => $user], 201);
        } catch (\Exception $e) {
            return response()->json(['message' => 'Registration failed', 'error' => $e->getMessage()], 500);
        }
    }
}
```

**Perbedaan dengan Repository Pattern:**
| Fitur          | Repository Pattern                                  | Service Pattern                                    |
| :------------- | :-------------------------------------------------- | :------------------------------------------------- |
| **Fokus Utama**| Abstraksi lapisan persistensi data (CRUD)           | Koordinasi logika bisnis kompleks                  |
| **Tanggung Jawab** | Interaksi dengan satu entitas/model di database | Mengatur dan mengkoordinasikan operasi antar entitas/repository/komponen lain |
| **Ketergantungan** | Bergantung pada model Eloquent (atau ORM lain) | Bergantung pada satu atau lebih Repository, Mailer, Event Dispatcher, dll. |
| **Tingkat Abstraksi** | Lapisan data (data access layer)                  | Lapisan bisnis (business logic layer)              |
| **Tingkat Granularitas** | Lebih rendah (operasi data spesifik)             | Lebih tinggi (operasi bisnis end-to-end)          |

**Kesimpulan:**
Repository Pattern berfokus pada bagaimana data disimpan dan diambil. Service Pattern berfokus pada *apa* yang dilakukan dengan data tersebut dan bagaimana berbagai operasi bisnis dikoordinasikan. Keduanya sering digunakan bersamaan untuk membangun aplikasi yang terstruktur dengan baik. Service menggunakan Repository untuk melakukan operasi data, sementara Controller menggunakan Service untuk menjalankan logika bisnis.

### 23\. Bagaimana cara menangani error dan exception di Laravel? Jelaskan prosesnya dan bagaimana kamu bisa membuat custom error pages.

Penanganan error dan exception di Laravel adalah fitur yang robust, menyediakan cara yang terpusat untuk mengelola kesalahan dalam aplikasi.

**Proses Penanganan Error dan Exception di Laravel:**

1.  **Throwing Exceptions:**
    Ketika terjadi kesalahan dalam kode Anda (misalnya, data tidak ditemukan, validasi gagal, atau ada masalah eksternal), Anda dapat melemparkan (`throw`) sebuah `Exception`. Laravel secara otomatis akan menangkap banyak exception umum (seperti `ModelNotFoundException` atau `ValidationException`). Anda juga dapat melemparkan custom exception.

    ```php
    // Contoh throwing exception
    if (!$user) {
        throw new \App\Exceptions\UserNotFoundException('Pengguna tidak ditemukan.');
    }
    ```

2.  **`App\Exceptions\Handler.php`:**
    Ini adalah jantung dari penanganan exception di Laravel. Semua exception yang tidak ditangkap secara eksplisit oleh blok `try-catch` Anda akan diteruskan ke kelas `Handler`. Kelas ini memiliki dua metode utama:

      * **`register()`:** Digunakan untuk mendaftarkan callback penanganan exception khusus. Anda bisa mendaftarkan callback untuk tipe exception tertentu, sehingga Anda dapat menulis logika kustom untuk menanganinya, seperti logging, mengirim notifikasi, atau mengembalikan respons HTTP tertentu.
        ```php
        // app/Exceptions/Handler.php
        public function register(): void
        {
            $this->reportable(function (Throwable $e) {
                // Contoh: Kirim notifikasi ke Slack jika terjadi kesalahan server
                if ($e instanceof \App\Exceptions\CustomServerException) {
                    // Logic untuk mengirim notifikasi
                }
            });

            $this->renderable(function (UserNotFoundException $e, Request $request) {
                if ($request->is('api/*')) {
                    return response()->json([
                        'message' => 'Resource not found.'
                    ], 404);
                }
            });
        }
        ```
      * **`report()`:** Digunakan untuk melaporkan exception. Secara default, ini akan mengirim exception ke log aplikasi Anda. Anda bisa menyesuaikan ini untuk mengirim laporan ke layanan eksternal seperti Sentry, Bugsnag, atau Slack.
      * **`render()`:** Digunakan untuk merender respons HTTP yang dikembalikan ke browser ketika exception terjadi. Anda bisa menyesuaikannya untuk mengembalikan view kustom, JSON, atau redirect, tergantung pada tipe exception dan jenis permintaan (web/API).

3.  **`config/app.php` (`debug`):**
    Pengaturan `APP_DEBUG` di file `.env` (dan dibaca di `config/app.php`) mengontrol apakah detail error akan ditampilkan kepada pengguna.

      * `APP_DEBUG=true` (development): Menampilkan stack trace lengkap dan detail error, sangat berguna untuk debugging.
      * `APP_DEBUG=false` (production): Menyembunyikan detail error sensitif dan hanya menampilkan pesan error generik atau custom error page.

4.  **Error Pages (Halaman Kesalahan):**
    Laravel memiliki fitur bawaan untuk menampilkan halaman error kustom berdasarkan kode status HTTP. Ketika sebuah exception di-render dan menghasilkan kode status HTTP tertentu (misalnya 404, 500, 403), Laravel akan mencari file Blade yang sesuai di direktori `resources/views/errors`.

**Cara Membuat Custom Error Pages:**

Anda dapat membuat file Blade di direktori `resources/views/errors` dengan nama sesuai kode status HTTP:

  * `resources/views/errors/404.blade.php`: Untuk kesalahan "Not Found".
  * `resources/views/errors/500.blade.php`: Untuk kesalahan server internal.
  * `resources/views/errors/403.blade.php`: Untuk kesalahan "Forbidden".
  * `resources/views/errors/419.blade.php`: Untuk kesalahan CSRF token expired.
  * Dan seterusnya untuk kode status HTTP lainnya.

**Contoh `resources/views/errors/404.blade.php`:**

```blade
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Halaman Tidak Ditemukan</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            display: flex;
            justify-content: center;
            align-items: center;
            min-height: 100vh;
            background-color: #f8f8f8;
            color: #333;
            text-align: center;
        }
        .container {
            padding: 20px;
            background-color: #fff;
            border-radius: 8px;
            box-shadow: 0 2px 10px rgba(0,0,0,0.1);
        }
        h1 {
            font-size: 4em;
            color: #e74c3c;
            margin-bottom: 10px;
        }
        p {
            font-size: 1.2em;
        }
        a {
            color: #3498db;
            text-decoration: none;
        }
        a:hover {
            text-decoration: underline;
        }
    </style>
</head>
<body>
    <div class="container">
        <h1>404</h1>
        <p>Maaf, halaman yang Anda cari tidak dapat ditemukan.</p>
        <p><a href="{{ url('/') }}">Kembali ke Beranda</a></p>
    </div>
</body>
</html>
```

Ketika Laravel menemukan respons dengan status 404 (misalnya, jika sebuah route tidak ditemukan atau Anda secara manual mengembalikan `abort(404)`), ia akan secara otomatis menampilkan halaman ini jika `APP_DEBUG` adalah `false`.

Untuk penanganan exception yang lebih spesifik atau kustomisasi respons berdasarkan `Request`, Anda harus menggunakan metode `renderable()` di `App\Exceptions\Handler.php`.

### 24\. Apa itu Design Pattern? Sebutkan minimal 3 Design Pattern yang sering digunakan dalam pengembangan web dengan Laravel dan jelaskan singkat fungsinya.

**Design Pattern (Pola Desain):**
Design Pattern adalah solusi umum dan dapat digunakan kembali untuk masalah yang sering muncul dalam desain perangkat lunak. Mereka adalah deskripsi tentang bagaimana memecahkan masalah umum. Mereka bukan kode yang siap pakai yang dapat langsung Anda colokkan ke aplikasi Anda, melainkan templat atau panduan yang dapat disesuaikan untuk kebutuhan spesifik Anda. Tujuan utamanya adalah untuk meningkatkan reusabilitas, fleksibilitas, dan kemudahan pemeliharaan kode.

**Minimal 3 Design Pattern yang Sering Digunakan dalam Pengembangan Web dengan Laravel:**

1.  **MVC (Model-View-Controller) - Architectural Pattern:**

      * **Fungsi:** Laravel sendiri dibangun di atas pola arsitektur MVC. MVC memisahkan aplikasi menjadi tiga komponen utama:
          * **Model:** Merepresentasikan data dan logika bisnis terkait data (berinteraksi dengan database). Di Laravel, ini adalah Eloquent Models.
          * **View:** Merepresentasikan antarmuka pengguna (UI) dan apa yang dilihat oleh pengguna. Di Laravel, ini adalah Blade templates.
          * **Controller:** Bertindak sebagai perantara antara Model dan View. Ia menerima input dari pengguna, mengolahnya (memanggil Model untuk data atau logika bisnis), dan kemudian mengembalikan respons (biasanya menampilkan View).
      * **Manfaat:** Memisahkan kekhawatiran (`separation of concerns`), membuat kode lebih terorganisir, mudah diuji, dan dipelihara.

2.  **Singleton Pattern - Creational Pattern:**

      * **Fungsi:** Memastikan bahwa sebuah kelas hanya memiliki satu instance (objek) di seluruh aplikasi dan menyediakan titik akses global ke instance tersebut.
      * **Di Laravel:** Service Container Laravel secara ekstensif menggunakan Singleton untuk binding. Ketika Anda mendaftarkan sebuah binding sebagai `singleton()` di Service Provider, Laravel akan membuat instance kelas tersebut hanya sekali dan menggunakannya kembali di setiap injeksi dependensi berikutnya. Contoh internal di Laravel adalah `App` (Service Container itu sendiri), `Request`, `Response`, dan banyak lagi yang di-resolve sebagai singleton.
      * **Manfaat:** Menghemat memori, mengelola sumber daya bersama (misalnya, koneksi database, konfigurasi), dan memastikan konsistensi.

3.  **Factory Method Pattern - Creational Pattern:**

      * **Fungsi:** Menyediakan antarmuka untuk membuat objek di superclass, tetapi memungkinkan subclass untuk mengubah tipe objek yang akan dibuat. Ini mendelegasikan instansiasi objek ke subclass.
      * **Di Laravel:**
          * **Factories Eloquent:** Fitur `factory()` di Laravel adalah contoh paling jelas. Anda mendefinisikan "factory" untuk sebuah model, yang kemudian dapat digunakan untuk membuat instance model tersebut dengan data dummy untuk pengujian atau seeding database.
            ```php
            // Database Seeder atau Test
            \App\Models\User::factory()->count(50)->create();
            ```
            Di sini, `User::factory()` adalah metode pabrik yang mengembalikan instance `UserFactory`, yang kemudian digunakan untuk membuat objek `User`.
          * **Facades:** Walaupun bukan Factory murni, Facades Laravel menggunakan Service Container untuk "membuat" instance di belakang layar saat pertama kali diakses, yang dapat dianggap sebagai semacam Factory.
      * **Manfaat:** Memisahkan kode instansiasi objek dari kode klien, membuat sistem lebih fleksibel dan mudah diperluas karena logika pembuatan objek dapat diubah tanpa memengaruhi kode yang menggunakannya.

**Pola Lain yang Sering Terlihat di Laravel:**

  * **Strategy Pattern:** Digunakan oleh driver cache, driver queue, driver filesystem, dll., di mana Anda dapat dengan mudah menukar implementasi (misalnya, dari Redis ke database untuk queue) tanpa mengubah kode yang menggunakannya.
  * **Observer Pattern:** Digunakan oleh Event/Listener system di Laravel, di mana objek (listener) "mengamati" objek lain (event) dan bereaksi ketika event terjadi.
  * **Dependency Injection (DI) & Inversion of Control (IoC):** Meskipun bukan Design Pattern tunggal, ini adalah prinsip desain fundamental yang diimplementasikan oleh Service Container Laravel.

### 25\. Jelaskan perbedaan antara Artisan Command, Custom PHP Class, dan Job di Laravel. Kapan masing-masing akan digunakan?

Ketiga komponen ini memiliki peran yang berbeda dalam arsitektur aplikasi Laravel, meskipun kadang-kadang dapat saling tumpang tindih dalam fungsionalitas.

**1. Artisan Command:**

  * **Definisi:** Ini adalah script yang dijalankan dari baris perintah (CLI - Command Line Interface) menggunakan `php artisan`. Mereka adalah cara yang ideal untuk menjalankan tugas-tugas administratif, pemeliharaan, atau operasi sekali pakai yang tidak perlu berinteraksi langsung dengan permintaan HTTP pengguna.
  * **Contoh Penggunaan:**
      * **Migrasi Database:** `php artisan migrate`
      * **Membuat komponen:** `php artisan make:controller`, `php artisan make:model`
      * **Membersihkan cache:** `php artisan cache:clear`, `php artisan view:clear`
      * **Tugas pemeliharaan harian/mingguan:** Mengirim laporan email, membersihkan data lama, mengindeks ulang pencarian (jika tidak cocok untuk queue).
      * **Script CLI kustom:** Misalnya, `php artisan import:data` untuk mengimpor data dari file CSV.
  * **Kapan Digunakan:**
      * Ketika Anda perlu menjalankan tugas secara manual dari terminal.
      * Ketika tugas perlu dijadwalkan secara berkala menggunakan Cron Job (misalnya, `php artisan schedule:run`).
      * Ketika tugas melibatkan logika bisnis yang kompleks tetapi tidak perlu dijalankan sebagai bagian dari permintaan web langsung.
      * Untuk tugas yang mungkin memakan waktu lama dan tidak perlu segera memberikan respons ke pengguna.
  * **Lingkup:** Biasanya berinteraksi dengan seluruh aplikasi, bukan spesifik untuk satu permintaan web.

**2. Custom PHP Class (non-Artisan Command, non-Job):**

  * **Definisi:** Ini adalah kelas PHP biasa yang Anda buat untuk mengorganisir logika bisnis atau fungsionalitas tertentu dalam aplikasi Anda. Mereka tidak secara otomatis terdaftar di Service Container sebagai Job atau Command, tetapi dapat di-instantiate atau di-resolve oleh Service Container.
  * **Contoh Penggunaan:**
      * **Service Classes:** (seperti yang dijelaskan di soal 22) untuk mengkapsulasi logika bisnis kompleks yang dapat digunakan di berbagai tempat.
      * **Repository Classes:** Untuk mengabstraksi interaksi database.
      * **Helper Classes:** Untuk fungsionalitas utilitas umum.
      * **Form Request Classes:** Untuk validasi input.
      * **Middleware Classes:** Untuk memfilter permintaan HTTP.
      * **Trait:** Untuk berbagi fungsionalitas di antara beberapa kelas.
  * **Kapan Digunakan:**
      * Sebagai bagian dari logika inti aplikasi yang diakses oleh controller, event listener, job, command, atau bagian lain dari aplikasi.
      * Ketika Anda perlu mengorganisir kode ke dalam unit-unit yang dapat digunakan kembali dan diuji.
      * Untuk logika yang tidak spesifik untuk CLI atau background queue, tetapi merupakan bagian dari alur kerja normal aplikasi.
  * **Lingkup:** Tergantung di mana mereka diinjeksi atau dipanggil.

**3. Job (Queueable Job):**

  * **Definisi:** Ini adalah kelas PHP yang mengimplementasikan interface `ShouldQueue`. Mereka dirancang untuk dijalankan secara asinkron di latar belakang oleh "worker" antrian. Setelah di-dispatch, job akan dimasukkan ke dalam antrian dan diproses ketika worker antrian tersedia.
  * **Contoh Penggunaan:**
      * **Pengiriman Email:** Mengirim email verifikasi, email notifikasi, atau email massal.
      * **Pemrosesan Gambar:** Resize, watermark, atau upload gambar ke layanan penyimpanan awan.
      * **Import/Export Data:** Memproses file CSV besar.
      * **Integrasi Pihak Ketiga:** Memanggil API eksternal yang mungkin lambat atau rawan kegagalan sementara.
      * **Pembaruan Status:** Memperbarui status pesanan setelah pembayaran diproses oleh gateway eksternal.
  * **Kapan Digunakan:**
      * Ketika Anda memiliki tugas yang memakan waktu (lebih dari beberapa ratus milidetik) dan tidak perlu memberikan respons instan kepada pengguna.
      * Ketika tugas dapat gagal dan perlu dicoba ulang (`retry`).
      * Untuk tugas yang akan memblokir respons HTTP jika dijalankan secara sinkron.
      * Untuk meningkatkan responsivitas aplikasi.
  * **Lingkup:** Dipicu oleh permintaan web (atau command), tetapi dieksekusi secara terpisah di latar belakang.

**Ringkasan Perbedaan dan Kapan Digunakan:**
| Tipe Komponen       | Digunakan Untuk                               | Kapan Digunakan                                        |
| :------------------ | :-------------------------------------------- | :----------------------------------------------------- |
| **Artisan Command** | Tugas CLI, administratif, pemeliharaan, sekali jalan, atau terjadwal | Operasi yang dijalankan dari terminal atau via cron job, tidak interaktif dengan browser. |
| **Custom PHP Class** | Mengorganisir logika bisnis, helper, utilitas | Bagian inti dari aplikasi yang digunakan oleh controller, job, command, dll. (fungsi umum). |
| **Job (Queueable)** | Tugas berat, lambat, atau asinkron          | Operasi yang tidak perlu respons instan dan dapat dijalankan di latar belakang untuk meningkatkan performa. |

### 26\. Jelaskan bagaimana routing bekerja di Laravel. Apa perbedaan antara route web dan route API?

**Bagaimana Routing Bekerja di Laravel:**

Routing di Laravel adalah cara untuk memetakan URL ke kode yang akan dieksekusi ketika URL tersebut diakses. Laravel menyediakan cara yang ekspresif dan fleksibel untuk mendefinisikan rute.

**Proses Umum:**

1.  **File Route:** Definisi rute biasanya berada di direktori `routes/`.
      * `routes/web.php`: Untuk rute aplikasi web yang berinteraksi dengan browser, biasanya membutuhkan sesi dan CSRF protection.
      * `routes/api.php`: Untuk rute API stateless yang biasanya mengembalikan JSON dan sering menggunakan token-based authentication.
      * `routes/console.php`: Untuk rute Artisan command.
      * `routes/channels.php`: Untuk rute broadcast event.
2.  **Permintaan Masuk:** Ketika permintaan HTTP masuk ke aplikasi Laravel, `public/index.php` adalah file pertama yang di-load.
3.  **Bootstrap Laravel:** `index.php` mem-bootstrap Laravel application, termasuk memuat Service Container dan semua Service Provider.
4.  **Router Service Provider:** Laravel secara otomatis memuat `RouteServiceProvider` (dari `app/Providers/RouteServiceProvider.php`). Service provider ini bertanggung jawab untuk memuat semua file rute Anda.
5.  **Pencocokan Rute:** Router Laravel akan mencoba mencocokkan URL permintaan masuk dengan definisi rute yang ada di file-file rute. Pencocokan terjadi berdasarkan:
      * **HTTP Method:** GET, POST, PUT, PATCH, DELETE.
      * **URI:** Pola URL (misalnya `/users`, `/users/{id}`).
6.  **Eksekusi Callback/Controller:** Setelah rute cocok, router akan mengeksekusi callback (closure) yang terkait dengan rute tersebut, atau memanggil metode pada Controller yang ditentukan.
7.  **Middleware:** Sebelum atau sesudah eksekusi logika utama, permintaan akan melewati middleware yang ditentukan pada rute atau grup rute.
8.  **Respons:** Kode akan mengembalikan respons HTTP (HTML, JSON, redirect, dll.) ke browser pengguna.

**Contoh Definisi Rute:**

```php
// routes/web.php
use App\Http\Controllers\HomeController;
use App\Http\Controllers\PostController;

Route::get('/', function () {
    return view('welcome');
});

Route::get('/home', [HomeController::class, 'index'])->name('home');

Route::resource('posts', PostController::class); // Contoh resource route

// routes/api.php
use App\Http\Controllers\Api\UserController;

Route::get('/users', [UserController::class, 'index']);
Route::post('/users', [UserController::class, 'store']);
```

**Perbedaan antara Route Web dan Route API:**

| Fitur             | Route Web (`routes/web.php`)                     | Route API (`routes/api.php`)                           |
| :---------------- | :----------------------------------------------- | :----------------------------------------------------- |
| **Utama Digunakan Untuk** | Aplikasi web tradisional (render HTML)         | API yang mengembalikan JSON/XML (untuk SPA, mobile, dll.) |
| **Middleware Default** | Group `web` (`StartSession`, `ShareErrorsFromSession`, `VerifyCsrfToken`, `SubstituteBindings`, dll.) | Group `api` (`SubstituteBindings`, `ThrottleRequests`, `AuthenticateSession` - optional). Defaultnya lebih ringan. |
| **Stateful vs. Stateless** | Umumnya **stateful** (menggunakan sesi/cookie) | Umumnya **stateless** (menggunakan token-based auth) |
| **CSRF Protection** | **Diaktifkan** secara default untuk POST/PUT/PATCH/DELETE | **Tidak diaktifkan** secara default (API tidak menggunakan CSRF token) |
| **Session State** | **Menggunakan** session state                     | **Tidak menggunakan** session state default (bisa diaktifkan jika perlu) |
| **URL Prefix** | Tidak ada prefix default (misal: `/dashboard`) | Prefix `/api` secara default (misal: `/api/users`) |
| **Respons Umum** | Mengembalikan View (HTML) atau Redirect        | Mengembalikan JSON atau data terstruktur lainnya       |

**Kapan Menggunakan Masing-masing:**

  * **Route Web:** Ketika Anda membangun aplikasi web di mana pengguna berinterinteraksi langsung dengan antarmuka yang dirender oleh server (menggunakan Blade templates), dan Anda memerlukan fitur seperti sesi, CSRF protection, dan autentikasi berbasis cookie.
  * **Route API:** Ketika Anda membangun backend untuk Single Page Application (SPA), aplikasi mobile, atau integrasi dengan layanan lain yang berkomunikasi melalui JSON dan menggunakan token (seperti OAuth, JWT, atau Sanctum) untuk autentikasi dan otorisasi.

### 27\. Bagaimana cara melakukan validasi input di Laravel? Jelaskan setidaknya dua metode yang berbeda.

Laravel menyediakan cara yang sangat fleksibel dan kuat untuk memvalidasi data yang masuk ke aplikasi Anda. Ada beberapa metode utama:

**1. Menggunakan Metode `validate()` pada Objek `Request` (atau Helper `validate()`):**
Ini adalah metode paling umum dan cepat untuk validasi sederhana hingga menengah, sering digunakan langsung di dalam controller.

  * **Cara Kerja:** Anda memanggil metode `validate()` pada instance `Illuminate\Http\Request`. Anda menyediakan array aturan validasi. Jika validasi gagal, Laravel akan secara otomatis mengarahkan kembali pengguna ke halaman sebelumnya (untuk permintaan web) atau mengembalikan respons JSON dengan pesan error (untuk permintaan API) dengan status 422 (Unprocessable Entity).
  * **Contoh (di Controller):**
    ```php
    <?php

    namespace App\Http\Controllers;

    use Illuminate\Http\Request;
    use App\Models\Post;

    class PostController extends Controller
    {
        public function store(Request $request)
        {
            // Validasi input
            $validatedData = $request->validate([
                'title' => 'required|string|max:255',
                'content' => 'required|string',
                'category_id' => 'required|exists:categories,id', // Memastikan category_id ada di tabel categories
                'status' => ['required', 'in:draft,published'], // Array untuk multiple rules
                'image' => 'nullable|image|mimes:jpeg,png,jpg,gif|max:2048', // Untuk upload file
            ]);

            // Jika validasi lolos, kode di bawah akan dieksekusi
            $post = Post::create($validatedData);

            return response()->json(['message' => 'Post created successfully', 'post' => $post], 201);
        }

        public function update(Request $request, Post $post)
        {
            $validatedData = $request->validate([
                'title' => 'sometimes|required|string|max:255', // 'sometimes' untuk opsional update
                'content' => 'sometimes|required|string',
                // ... aturan validasi lainnya
            ]);

            $post->update($validatedData);

            return response()->json(['message' => 'Post updated successfully', 'post' => $post]);
        }
    }
    ```
  * **Manfaat:** Cepat dan mudah diimplementasikan, terutama untuk controller yang sederhana. Menangani pengalihan dan respons JSON secara otomatis.
  * **Kelemahan:** Jika aturan validasi sangat banyak atau kompleks, controller bisa menjadi gemuk. Tidak cocok untuk reusabilitas validasi di berbagai tempat.

**2. Menggunakan Form Request Classes:**
Ini adalah metode yang disarankan untuk validasi yang lebih kompleks, besar, atau yang perlu digunakan kembali di berbagai controller atau metode.

  * **Cara Kerja:** Anda membuat kelas Form Request terpisah yang mewarisi dari `Illuminate\Foundation\Http\FormRequest`. Kelas ini memiliki dua metode utama:
      * `authorize()`: Untuk menentukan apakah pengguna saat ini diotorisasi untuk membuat permintaan ini. Jika mengembalikan `false`, akan mengembalikan respons 403 (Forbidden).
      * `rules()`: Mengembalikan array aturan validasi, sama seperti metode `validate()` pada objek `Request`.
  * **Contoh:**
    1.  **Buat Form Request:**
        ```bash
        php artisan make:request StorePostRequest
        ```
    2.  **Edit `app/Http/Requests/StorePostRequest.php`:**
        ```php
        <?php

        namespace App\Http\Requests;

        use Illuminate\Foundation\Http\FormRequest;

        class StorePostRequest extends FormRequest
        {
            /**
             * Determine if the user is authorized to make this request.
             */
            public function authorize(): bool
            {
                // Misalnya, hanya admin atau user yang login yang bisa membuat post
                return auth()->check(); // Atau auth()->user()->isAdmin();
            }

            /**
             * Get the validation rules that apply to the request.
             *
             * @return array<string, \Illuminate\Contracts\Validation\ValidationRule|array<mixed>|string>
             */
            public function rules(): array
            {
                return [
                    'title' => 'required|string|min:5|max:255|unique:posts,title', // Tambah unique rule
                    'content' => 'required|string',
                    'category_id' => 'required|integer|exists:categories,id',
                    'status' => ['required', 'in:draft,published'],
                    'image' => 'nullable|image|mimes:jpeg,png,jpg,gif|max:2048',
                ];
            }

            /**
             * Get the error messages for the defined validation rules.
             */
            public function messages(): array
            {
                return [
                    'title.required' => 'Judul wajib diisi.',
                    'title.unique' => 'Judul ini sudah ada, gunakan judul lain.',
                    'content.required' => 'Konten postingan tidak boleh kosong.',
                    'category_id.exists' => 'Kategori yang dipilih tidak valid.',
                    // ... pesan kustom lainnya
                ];
            }
        }
        ```
    3.  **Gunakan di Controller:**
        ```php
        <?php

        namespace App\Http\Controllers;

        use App\Http\Requests\StorePostRequest; // Import Form Request
        use App\Models\Post;

        class PostController extends Controller
        {
            public function store(StorePostRequest $request) // Injeksi Form Request di sini
            {
                // Jika validasi (dan otorisasi) lolos, kode di bawah akan dieksekusi.
                // $request->validated() akan mengembalikan hanya data yang sudah tervalidasi.
                $post = Post::create($request->validated());

                return response()->json(['message' => 'Post created successfully', 'post' => $post], 201);
            }
        }
        ```
  * **Manfaat:**
      * **Pemisahan Kekhawatiran:** Memisahkan logika validasi dan otorisasi dari controller.
      * **Reusabilitas:** Satu Form Request dapat digunakan di beberapa controller atau metode.
      * **Kode Bersih:** Controller tetap ramping dan fokus pada logika bisnis intinya.
      * **Pesan Kustom:** Mudah untuk mendefinisikan pesan error kustom.
  * **Kelemahan:** Membutuhkan pembuatan file baru untuk setiap set aturan validasi.

**Pilihan Lain (untuk kasus sangat spesifik):**

  * **`Validator::make()`:** Menggunakan facade `Validator` secara manual. Memberikan kontrol penuh atas proses validasi, cocok untuk validasi yang sangat spesifik atau untuk kasus di mana Anda ingin menangani error secara manual (misalnya, tanpa redirect otomatis).
    ```php
    use Illuminate\Support\Facades\Validator;

    $validator = Validator::make($request->all(), [
        'name' => 'required|max:255',
        'email' => 'required|email|unique:users',
    ]);

    if ($validator->fails()) {
        return redirect('post/create')
                    ->withErrors($validator)
                    ->withInput();
    }
    ```

### 28\. Jelaskan perbedaan antara `find()`, `where()`, dan `first()` di Eloquent. Kapan kamu akan menggunakan masing-masing?

Ketiga metode ini digunakan untuk mengambil data dari database menggunakan Eloquent ORM di Laravel, tetapi dengan tujuan dan cara kerja yang berbeda.

**1. `find()`:**

  * **Fungsi:** Digunakan untuk mengambil sebuah model tunggal berdasarkan kunci primernya (`id`). Jika model dengan ID yang diberikan tidak ditemukan, ia akan mengembalikan `null`.
  * **Cara Kerja:** Ini adalah shortcut yang efisien untuk `where('id', '=', $id)->first()`.
  * **Contoh:**
    ```php
    use App\Models\User;

    // Mengambil user dengan ID 1
    $user = User::find(1);

    if ($user) {
        echo $user->name;
    } else {
        echo "User not found.";
    }

    // Mengambil beberapa user berdasarkan array ID
    $users = User::find([1, 2, 3]); // Mengembalikan Collection of Users
    ```
  * **Kapan Digunakan:**
      * Ketika Anda yakin ingin mencari model berdasarkan kunci primernya.
      * Untuk mengambil satu record spesifik yang ID-nya sudah Anda ketahui.

**2. `where()`:**

  * **Fungsi:** Digunakan untuk menambahkan kondisi (clause) `WHERE` ke query database. Metode ini mengembalikan instance `Illuminate\Database\Eloquent\Builder`, yang berarti Anda dapat melanjutkan chaining metode query lainnya (seperti `orderBy()`, `limit()`, `get()`, `first()`, dll.).
  * **Cara Kerja:** Ini adalah dasar untuk membangun query yang kompleks. Anda dapat memanggil `where()` berkali-kali untuk menambahkan beberapa kondisi.
  * **Contoh:**
    ```php
    use App\Models\Product;

    // Mengambil semua produk yang statusnya 'published'
    $publishedProducts = Product::where('status', 'published')->get(); // Mengembalikan Collection

    // Mengambil produk dengan harga lebih dari 100 dan kategori 5
    $expensiveProductsInCategory = Product::where('price', '>', 100)
                                        ->where('category_id', 5)
                                        ->get();

    // Mengambil satu produk yang memiliki nama 'Laptop Gaming' (jika ada beberapa, akan mengembalikan yang pertama ditemukan)
    $laptop = Product::where('name', 'Laptop Gaming')->first();
    ```
  * **Kapan Digunakan:**
      * Ketika Anda ingin mengambil satu atau lebih record berdasarkan kriteria selain kunci primer.
      * Ketika Anda perlu membangun query yang lebih kompleks dengan berbagai kondisi, pengurutan, atau batasan.

**3. `first()`:**

  * **Fungsi:** Digunakan untuk mengambil **record tunggal pertama** yang cocok dengan kondisi query yang telah Anda bangun. Jika tidak ada record yang cocok, ia akan mengembalikan `null`.
  * **Cara Kerja:** Biasanya dipanggil setelah satu atau lebih metode `where()` untuk mengambil satu record yang paling sesuai.
  * **Contoh:**
    ```php
    use App\Models\User;

    // Mengambil user pertama dengan email tertentu
    $adminUser = User::where('email', 'admin@example.com')->first();

    if ($adminUser) {
        echo "Admin: " . $adminUser->name;
    } else {
        echo "Admin user not found.";
    }

    // Mengambil post terbaru
    $latestPost = Post::orderBy('created_at', 'desc')->first();
    ```
  * **Kapan Digunakan:**
      * Ketika Anda hanya membutuhkan satu record dari hasil query, meskipun query tersebut mungkin akan mengembalikan banyak record jika Anda memanggil `get()`.
      * Sering digunakan bersama `where()` untuk mendapatkan record spesifik berdasarkan kondisi yang tidak harus kunci primer.

**Ringkasan Perbedaan:**

| Metode      | Fungsi Utama                             | Tipe Kembalian Jika Ditemukan  | Tipe Kembalian Jika Tidak Ditemukan | Kapan Digunakan                                         |
| :---------- | :--------------------------------------- | :----------------------------- | :---------------------------------- | :------------------------------------------------------ |
| `find($id)` | Mencari 1 record berdasarkan PRIMARY KEY | Model instance (atau Collection jika array ID) | `null`                               | Mengambil record tunggal yang ID-nya sudah diketahui.   |
| `where()`   | Menambahkan kondisi query WHERE          | `Eloquent\Builder` (query builder) | N/A (belum dieksekusi)              | Membangun query, harus diakhiri dengan `get()`, `first()`, dll. |
| `first()`   | Mengambil 1 record PERTAMA dari hasil query | Model instance                   | `null`                               | Mengambil record tunggal berdasarkan kriteria kompleks. |

### 29\. Buatlah sebuah query builder di Laravel untuk mengambil data `orders` yang memiliki `status` 'completed', dibuat dalam 30 hari terakhir, dan diurutkan berdasarkan `total_amount` secara menurun. Sertakan relasi `user` dan `order_items` (with 'product').

Tentu, ini adalah query builder Eloquent di Laravel untuk skenario yang Anda minta:

```php
<?php

namespace App\Http\Controllers;

use App\Models\Order;
use Carbon\Carbon; // Pastikan Carbon diimport
use Illuminate\Http\Request;

class OrderController extends Controller
{
    public function getCompletedOrders(Request $request)
    {
        $orders = Order::with(['user', 'orderItems.product']) // Eager load 'user' dan 'orderItems', serta 'product' di dalam 'orderItems'
            ->where('status', 'completed') // Filter berdasarkan status 'completed'
            ->where('created_at', '>=', Carbon::now()->subDays(30)) // Filter untuk 30 hari terakhir
            ->orderBy('total_amount', 'desc') // Urutkan berdasarkan total_amount secara menurun
            ->get(); // Eksekusi query dan ambil hasilnya

        // Jika Anda ingin pagination, ganti ->get() dengan ->paginate(10) atau sesuai kebutuhan
        // $orders = Order::with(['user', 'orderItems.product'])
        //     ->where('status', 'completed')
        //     ->where('created_at', '>=', Carbon::now()->subDays(30))
        //     ->orderBy('total_amount', 'desc')
        //     ->paginate(10);

        return response()->json($orders);
    }
}

// Asumsi Model (contoh sederhana):

// app/Models/Order.php
// class Order extends Model
// {
//     public function user()
//     {
//         return $this->belongsTo(User::class);
//     }

//     public function orderItems()
//     {
//         return $this->hasMany(OrderItem::class);
//     }
// }

// app/Models/User.php
// class User extends Model
// {
//     protected $guarded = []; // Contoh untuk membolehkan mass assignment
// }

// app/Models/OrderItem.php
// class OrderItem extends Model
// {
//     public function order()
//     {
//         return $this->belongsTo(Order::class);
//     }

//     public function product()
//     {
//         return $this->belongsTo(Product::class);
//     }
// }

// app/Models/Product.php
// class Product extends Model
// {
//     protected $guarded = []; // Contoh
// }
```

**Penjelasan Query:**

1.  **`Order::with(['user', 'orderItems.product'])`**:

      * Ini adalah bagian **Eager Loading**.
      * `'user'`: Memberi tahu Eloquent untuk memuat data pengguna yang terkait dengan setiap pesanan (`Order hasMany User` atau `User belongsTo Order`). Ini akan menjalankan query terpisah untuk user tetapi jauh lebih efisien daripada Lazy Loading.
      * `'orderItems.product'`: Memberi tahu Eloquent untuk memuat `orderItems` yang terkait dengan setiap pesanan, dan di dalam setiap `orderItem`, muat juga `product` yang terkait. Ini dikenal sebagai "nested eager loading".

2.  **`->where('status', 'completed')`**:

      * Ini adalah bagian **Filtering**.
      * Menambahkan kondisi ke query yang hanya akan mengembalikan pesanan di mana kolom `status` bernilai `'completed'`.

3.  **`->where('created_at', '>=', Carbon::now()->subDays(30))`**:

      * Ini juga bagian **Filtering**.
      * Menggunakan `Carbon::now()->subDays(30)` untuk mendapatkan tanggal dan waktu 30 hari yang lalu dari waktu saat ini.
      * Kondisi ini memastikan bahwa hanya pesanan yang dibuat (berdasarkan kolom `created_at`) dalam 30 hari terakhir yang disertakan.

4.  **`->orderBy('total_amount', 'desc')`**:

      * Ini adalah bagian **Sorting**.
      * Mengurutkan hasil query berdasarkan kolom `total_amount` secara menurun (`desc`), yang berarti pesanan dengan `total_amount` terbesar akan muncul lebih dulu.

5.  **`->get()`**:

      * Ini adalah metode terminal yang mengeksekusi query database dan mengembalikan hasil sebagai `Illuminate\Database\Eloquent\Collection` dari objek `Order`.

**Tambahan (jika ingin Pagination):**

Jika dataset `orders` sangat besar, lebih baik menggunakan pagination untuk menghindari memuat semua data sekaligus ke memori. Anda bisa mengganti `.get()` dengan `.paginate($perPage)`:

```php
// ...
            ->paginate(10); // Akan mengembalikan 10 order per halaman
// ...
```

Ini akan secara otomatis menangani parameter `page` dari query string URL dan mengembalikan objek pagination yang berguna untuk API atau tampilan Blade.

### 30\. Jelaskan bagaimana kamu akan melakukan refactoring pada kode yang memiliki technical debt. Sebutkan langkah-langkah dan berikan contoh.

Refactoring adalah proses restrukturisasi kode yang ada tanpa mengubah perilaku eksternalnya, dengan tujuan meningkatkan kualitas non-fungsional dari kode tersebut (misalnya, keterbacaan, pemeliharaan, fleksibilitas). Technical debt mengacu pada "biaya" tambahan di masa depan karena keputusan desain yang buruk atau jalan pintas yang diambil di masa lalu.

**Langkah-langkah Refactoring pada Kode dengan Technical Debt:**

1.  **Identifikasi Technical Debt:**

      * **Code Smells:** Cari indikator umum masalah desain seperti:
          * **Long Methods/Classes:** Metode atau kelas yang terlalu panjang, melakukan terlalu banyak hal.
          * **Duplicate Code:** Bagian kode yang sama muncul di beberapa tempat.
          * **Large Class/God Object:** Sebuah kelas yang terlalu bertanggung jawab atas banyak hal.
          * **Feature Envy:** Sebuah metode yang lebih tertarik pada data kelas lain daripada datanya sendiri.
          * **Shotgun Surgery:** Perubahan kecil dalam fungsionalitas membutuhkan banyak perubahan di banyak file.
          * **Spaghetti Code:** Kode tanpa struktur yang jelas, banyak lompatan (`goto` atau kondisi bersarang yang kompleks).
      * **Kurangnya Tes:** Kode tanpa atau dengan tes yang buruk adalah kandidat utama untuk technical debt karena sulit untuk direfaktor dengan aman.
      * **Bug Berulang:** Bug yang terus muncul di area kode tertentu.
      * **Keluhan Pengembang:** Tim sering mengeluh tentang kesulitan memahami atau memodifikasi bagian kode tertentu.

2.  **Tulis Tes (jika belum ada):**

      * **Ini adalah langkah paling KRUSIAL.** Anda tidak bisa melakukan refactoring dengan aman tanpa mengetahui bahwa Anda tidak merusak fungsionalitas yang sudah ada.
      * Tulis tes unit atau tes integrasi untuk bagian kode yang akan Anda refaktor, pastikan tes tersebut mencakup perilaku eksternal dari kode tersebut.

3.  **Refactor dalam Langkah Kecil dan Bertahap:**

      * Jangan mencoba refactor seluruh modul sekaligus. Fokus pada satu "code smell" atau satu area kecil pada satu waktu.
      * Setelah setiap perubahan kecil, jalankan tes untuk memastikan tidak ada yang rusak.

4.  **Gunakan Teknik Refactoring Umum:**

      * **Extract Method:** Jika sebuah metode terlalu panjang, pisahkan bagian-bagiannya menjadi metode-metode yang lebih kecil dan lebih fokus.
      * **Extract Class:** Jika sebuah kelas melakukan terlalu banyak atau menangani beberapa tanggung jawab, pisahkan tanggung jawab tersebut ke kelas-kelas baru.
      * **Introduce Parameter Object:** Jika sebuah metode memiliki terlalu banyak parameter, bungkus parameter terkait ke dalam sebuah objek.
      * **Replace Conditional with Polymorphism:** Ganti blok `if/else if/else` atau `switch` yang besar dengan polymorphism untuk kode yang lebih mudah diperluas.
      * **Move Method/Field:** Pindahkan metode atau properti ke kelas yang lebih tepat jika mereka lebih sering digunakan di sana.
      * **Rename Method/Variable/Class:** Ubah nama elemen kode agar lebih deskriptif dan mencerminkan tujuannya.
      * **Decompose Conditional:** Sederhanakan pernyataan kondisional yang kompleks.

5.  **Verifikasi dengan Tes:**

      * Setelah setiap langkah refactoring, atau setelah serangkaian langkah kecil, jalankan kembali semua tes Anda. Ini adalah jaring pengaman Anda.

6.  **Review dan Iterate:**

      * Minta rekan tim untuk meninjau perubahan Anda. Diskusi dapat mengungkapkan area untuk perbaikan lebih lanjut atau masalah yang terlewat.
      * Refactoring adalah proses berulang. Anda mungkin perlu kembali ke langkah 1 setelah menyelesaikan satu siklus.

**Contoh Refactoring (Laravel):**

Misalkan Anda memiliki controller gemuk (`Fat Controller`) seperti ini:

**Kode Awal (Technical Debt: Long Method, Potential Duplicate Logic):**

```php
// app/Http/Controllers/OrderController.php
<?php

namespace App\Http\Controllers;

use App\Models\Order;
use App\Models\Product;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\DB;
use Illuminate\Support\Facades\Mail;
use App\Mail\OrderConfirmed;

class OrderController extends Controller
{
    public function store(Request $request)
    {
        // === Bagian 1: Validasi ===
        $request->validate([
            'customer_name' => 'required|string',
            'customer_email' => 'required|email',
            'items' => 'required|array',
            'items.*.product_id' => 'required|exists:products,id',
            'items.*.quantity' => 'required|integer|min:1',
        ]);

        // === Bagian 2: Logika Bisnis Kompleks (Order Creation) ===
        DB::beginTransaction();
        try {
            $totalAmount = 0;
            $order = Order::create([
                'customer_name' => $request->customer_name,
                'customer_email' => $request->customer_email,
                'status' => 'pending',
                'total_amount' => 0, // Akan diupdate nanti
            ]);

            foreach ($request->items as $itemData) {
                $product = Product::find($itemData['product_id']);
                if (!$product || $product->stock < $itemData['quantity']) {
                    DB::rollBack();
                    return response()->json(['message' => 'Stok tidak cukup atau produk tidak ditemukan.'], 400);
                }

                $order->orderItems()->create([
                    'product_id' => $product->id,
                    'quantity' => $itemData['quantity'],
                    'price' => $product->price,
                    'subtotal' => $product->price * $itemData['quantity'],
                ]);

                $product->decrement('stock', $itemData['quantity']);
                $totalAmount += $product->price * $itemData['quantity'];
            }

            $order->update(['total_amount' => $totalAmount]);

            // === Bagian 3: Notifikasi / Side Effects ===
            Mail::to($order->customer_email)->send(new OrderConfirmed($order));

            DB::commit();

            return response()->json(['message' => 'Pesanan berhasil dibuat!', 'order' => $order], 201);

        } catch (\Exception $e) {
            DB::rollBack();
            // Log error
            return response()->json(['message' => 'Gagal membuat pesanan.', 'error' => $e->getMessage()], 500);
        }
    }
}
```

**Refactoring Menggunakan Form Request dan Service Pattern:**

**Langkah 1: Ekstrak Validasi ke Form Request.**

```bash
php artisan make:request StoreOrderRequest
```

```php
// app/Http/Requests/StoreOrderRequest.php
<?php

namespace App\Http\Requests;

use Illuminate\Foundation\Http\FormRequest;

class StoreOrderRequest extends FormRequest
{
    public function authorize(): bool
    {
        return true; // Sesuaikan dengan logika otorisasi Anda
    }

    public function rules(): array
    {
        return [
            'customer_name' => 'required|string',
            'customer_email' => 'required|email',
            'items' => 'required|array',
            'items.*.product_id' => 'required|exists:products,id',
            'items.*.quantity' => 'required|integer|min:1',
        ];
    }
}
```

**Langkah 2: Ekstrak Logika Bisnis ke Service Class.**

```bash
php artisan make:service OrderService
```

```php
// app/Services/OrderService.php
<?php

namespace App\Services;

use App\Models\Order;
use App\Models\Product;
use Illuminate\Support\Facades\DB;
use Illuminate\Support\Facades\Mail;
use App\Mail\OrderConfirmed;
use App\Exceptions\InsufficientStockException; // Buat exception kustom jika perlu

class OrderService
{
    public function createOrder(array $orderData): Order
    {
        DB::beginTransaction();
        try {
            $totalAmount = 0;
            $order = Order::create([
                'customer_name' => $orderData['customer_name'],
                'customer_email' => $orderData['customer_email'],
                'status' => 'pending',
                'total_amount' => 0,
            ]);

            foreach ($orderData['items'] as $itemData) {
                $product = Product::find($itemData['product_id']);

                // Penanganan stok bisa lebih kompleks (misal, menggunakan pessimistic locking)
                if (!$product || $product->stock < $itemData['quantity']) {
                    throw new InsufficientStockException('Stok tidak cukup untuk produk: ' . ($product ? $product->name : 'Unknown Product'));
                }

                $order->orderItems()->create([
                    'product_id' => $product->id,
                    'quantity' => $itemData['quantity'],
                    'price' => $product->price,
                    'subtotal' => $product->price * $itemData['quantity'],
                ]);

                $product->decrement('stock', $itemData['quantity']);
                $totalAmount += $product->price * $itemData['quantity'];
            }

            $order->update(['total_amount' => $totalAmount]);

            Mail::to($order->customer_email)->send(new OrderConfirmed($order));

            DB::commit();

            return $order;

        } catch (\Exception $e) {
            DB::rollBack();
            // Anda bisa melempar ulang exception kustom atau log di sini
            throw $e; // Lempar ulang agar controller bisa menangani
        }
    }
}
```

**Langkah 3: Update Controller agar menggunakan Form Request dan Service.**

```php
// app/Http/Controllers/OrderController.php (Setelah Refactoring)
<?php

namespace App\Http\Controllers;

use App\Http\Requests\StoreOrderRequest; // Import Form Request
use App\Services\OrderService;           // Import Service Class
use Illuminate\Http\Request;
use App\Exceptions\InsufficientStockException; // Import custom exception jika dibuat

class OrderController extends Controller
{
    protected $orderService;

    public function __construct(OrderService $orderService)
    {
        $this->orderService = $orderService;
    }

    public function store(StoreOrderRequest $request)
    {
        try {
            // Validasi dan otorisasi sudah dihandle oleh StoreOrderRequest
            $order = $this->orderService->createOrder($request->validated());

            return response()->json(['message' => 'Pesanan berhasil dibuat!', 'order' => $order], 201);

        } catch (InsufficientStockException $e) {
            return response()->json(['message' => $e->getMessage()], 400);
        } catch (\Exception $e) {
            // Log error di sini atau di Exception Handler
            return response()->json(['message' => 'Gagal membuat pesanan.', 'error' => $e->getMessage()], 500);
        }
    }
}
```

**Manfaat Refactoring Ini:**

  * **Controller Lebih Ramping:** Metode `store` di controller sekarang jauh lebih ringkas dan fokus pada memanggil layanan dan mengembalikan respons.
  * **Keterpisahan Kekhawatiran:** Validasi berada di `StoreOrderRequest`, logika bisnis kompleks berada di `OrderService`.
  * **Reusabilitas:** `OrderService` dapat dipanggil dari mana saja (misalnya, Artisan command untuk membuat pesanan massal).
  * **Kemudahan Uji:** Baik `StoreOrderRequest` maupun `OrderService` sekarang lebih mudah diuji secara independen.
  * **Fleksibilitas:** Jika logika pembuatan pesanan berubah, Anda hanya perlu memodifikasi `OrderService`.# Tes-Teori-Solopos
