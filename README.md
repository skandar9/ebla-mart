Project overview:

Mobile app retail store..for sale electrical appliances.
It supports the provision of offers on products by the store owner, 
and the ability to evaluate products and add them to the favorites list by users in addition to make orders.

It contains:
- User Roles:
   The application supports multiple user roles, including regular users and administrators. Regular users can access and interact with the platform's content, while administrators have additional privileges to manage and moderate the platform.

- Authentication and Authorization:
   User authentication is handled using the Sanctum authentication middleware, ensuring that only authenticated users can access certain functionalities. The application also utilizes role-based authorization, where only users with the roles of "super_admin" or "admin" can perform specific actions.

- Content Organization:
   The platform organizes content into categories such as "Today," "Week," "Month," and "Year," allowing users to browse and discover the latest and most popular programming topics. Each topic is associated with a specific author, providing credit for contributions.

- Suggestion System:
   The application includes a suggestion feature that allows users to recommend related content to others. Users can suggest either a product or an offer, which are associated with specific IDs and types (e.g., Product::class and Offer::class). When a suggestion is made, a notification is triggered, providing an option to notify the user who made the suggestion or all users.

- Notifications:
   The platform incorporates a notification system for new suggestions. A notification is created when a new suggestion is made, with the option to send the notification to a specific user or all users. This feature helps keep users informed about relevant content suggestions.

- API Endpoints:
   The project provides several API endpoints to interact with the platform. These endpoints include retrieving all suggestions, creating new suggestions, retrieving a specific suggestion, and deleting a suggestion.



> :warning: **Warning:** This contents below ↓ contains just parts of my code.
>                        You can access my full project files by clone it from my GitLab repository
>                        (requires asking for my permissions  to grant you access to it):
>                        https://gitlab.com/skandar.s1998/aim 

## Contents
(contains descriptive parts of my code)

[Tables and relations](#tables-and-relations)

[Project actions and progress(graph)](#project-graph)

[Authentication](#authentication)

[Create product](#store-product)

[Add media to specific product](#product-media)

[Store new language functionality](#languages)

[Show all offers](#offers)

[Add a specific product to favorites](#add-to-favorite)

[Search method](#search)

[Relation between offers and products](#offer-product)


### **tables-and-relations**

![Logo](/images/tables.png)

For more details about the content of the tables <a href="/ebla.pdf" target="_blank">Click here</a>

[🔝 Back to contents](#contents)

### **project-graph**

This graph diagram represents the actions and progress for the project.

![App Logo](/graph_1.png)
![App Logo](/graph_2.png)

[🔝 Back to contents](#contents)

### **authentication**

app\Http\Controllers\AuthController.php:

The constructor, this function is part of a controller class and is responsible for setting up themiddleware for authentication using Laravel Sanctum.
This middleware ensures that the user is authenticated using the Sanctum authentication guardbefore accessing the methods.
The $this->middleware('auth:sanctum')->only(['logout', 'user']); line specifies that the'auth:sanctum' middleware should be applied only to the 'logout' and 'user' methods.

```php
    public function __construct()
    {
        $this->middleware('auth:api')->only(['logout']);
    }
```

```php
    public function register (Request $request) 
    {
        $request->validate([
            'name'       => ['required', 'string'],
            'email'      => ['required', 'string', 'email', 'unique:users'],  
            'password'   => ['required', 'string', 'min:6', 'confirmed'],
        ]);

        $user = User::create([
            'name'      => $request->name,
            'email'     => $request->email,
            'password'  => Hash::make($request->password),
            'role'      => 'user',
            'balance'   => 0,
        ]);

        $token = $user->createToken('Proxy App')->accessToken;
        
        return response()->json([
            'user' => new UserResource($user),
            'token' => $token,
        ], 200);
    }
```

```php
    public function login (Request $request) 
    {
       $request->validate([
            'email'      => ['required', 'string', 'email'],  
            'password'   => ['required', 'string'],
        ]);

        $user = User::where('email', $request->email)->first();

        if ($user) {
            if (Hash::check($request->password, $user->password)) {
                $token = $user->createToken('Mega Panel App')->accessToken;

                return response()->json([
                    'user' => new UserResource($user),
                    'token' => $token,
                ], 200);   
            }
        }
        
        return response()->json([
            'message' => 'email or password is incorrect.',
            'errors' => [
                'email' => ['email or password is incorrect.']
            ]
        ], 422);
    }
```

[🔝 Back to contents](#contents)

## store-product

`app\Http\Controllers\api\ProductController.php`

```php
public function store(Request $request)
{
    $request->validate([
        'code'            => ['required', 'string'],
        'title'           => ['required', 'array', LanguageMiddleware::rule()],
        'description'     => ['required', 'array', LanguageMiddleware::rule()],
        'brand_id'        => ['required', 'exists:brands,id,deleted_at,NULL'],
        'category_id'     => ['required', 'exists:categories,id,deleted_at,NULL'],
        'wholesale_price' => ['required', 'numeric', 'min:0'],
        'retail_price'    => ['required', 'numeric', 'min:0'],
        'quantity'        => ['required', 'numeric', 'min:0'],
        'available'       => ['boolean'],
        'notify'          => ['boolean'],
    ]);

    $product = Product::createWithTranslations([
        'code'            => $request->code,
        'title'           => $request->title,
        'description'     => $request->description,
        'brand_id'        => $request->brand_id,
        'category_id'     => $request->category_id,
        'wholesale_price' => $request->wholesale_price,
        'retail_price'    => $request->retail_price,
        'quantity'        => $request->quantity,
        'available'       => $request->available ?? 1,
    ]);

    if ($request->notify) {
        $notification = Notification::create_notification(
            'new_product',
            null,
            $product,
            []
        );
        $notification->send_to_all_users();
    }

    return response()->json(new ProductAdminResource($product), 200);
}
```

The `store` function in the `ProductController.php` file is responsible for storing a new product. It first validates the incoming request data based on certain rules. For example, the `code` field is required and must be a string. The `title` and `description` fields are also required and must be arrays, allowing for translations in different languages. The [LanguageMiddleware ](#language-middleware) is a custom validation rule that ensures the language-specific requirements for the `title` field.

After validating the request data, the function creates a new product with its translations using the `createWithTranslations` method of the `Product` model. The values for various attributes, such as `code`, `title`, `description`, `brand_id`, `category_id`, `wholesale_price`, `retail_price`, `quantity`, and `available`, are retrieved from the request.

If the `notify` field in the request is evaluated as true, the function creates a new notification. It utilizes the [create_notification method](#create_notification) from the [Notification Model](#notification-model),
The purpose of this notification is to inform users about the creation of a new product. The method accepts parameters notification type, a target user (null in this function), the related product, and additional data (an empty array in this function).

Finally, the function sends the created notification to all users by calling the `send_to_all_users` method on the notification. It then returns a JSON response containing the newly created product data using the `ProductAdminResource` resource.

This function is called from the following route defined in the `routes\api.php` file:

```php
Route::apiResource('products', LanguageController::class);
```

[🔝 Back to contents](#contents)

### **notification-model**

`app\Models\Notification.php`

```php
class Notification extends Model
{
    use HasFactory;

    protected $fillable = [
        'type',
        'source_type',
        'source_id',
        'target_type',
        'target_id',
        'data',
    ];

    public function source()
    {
        return $this->morphTo();
    }
    
    public function target()
    {
        return $this->morphTo();
    }

    public static function create_notification($type, $source, $target, $data)
    {
        return Notification::create([
            'type'        => $type,
            'source_type' => $source?get_class($source):null,
            'source_id'   => $source?->id,
            'target_type' => $target?get_class($target):null,
            'target_id'   => $target?->id,
            'data'        => json_encode($data),
        ]);
    }
    .
    .
    .
    public function send_to_all_users()
    {
        $users_ids = User::where('role','user')->where('verified', true)->pluck('id');
        $data = [];
        foreach($users_ids as $user_id)
            $data[$user_id] = ['read' => 0];
        $this->users()->attach($data);
        $this->send($users_ids);
    }
    .
    .
    .
    private function send($users_ids)
    {
        $message = [];
        $registration_ids = [];
        foreach(Language::$all_languages as $lang)
        {
            $message[$lang] = $this->get_message($lang);
            $registration_ids[$lang] = [];
        }
        
        $fcm_tokens = FCMToken::whereIn('user_id',$users_ids)->get();
        foreach($fcm_tokens as $fcm_token)
            $registration_ids[$fcm_token->language][] = $fcm_token->fcm_token;
    
        $key = env('FIREBASE_KEY','');

        foreach(Language::$all_languages as $lang)
        {
            if(count($registration_ids[$lang])>0)
                Http::withHeaders([
                    'Content-Type' => 'application/json',
                    'Authorization' => "key=$key",
                ])->post('https://fcm.googleapis.com/fcm/send',[
                    'registration_ids' => $registration_ids[$lang],
                    'notification' => [
                        'title' => 'Ebla Mart',
                        'body' => $message[$lang],
                    ]
                ]);
        }
    }

    public function get_message($language)
    {
        $params = [];
        if($this->type == "account_verification"){
            $params['user_name'] = $this->target->name;
        }

        return __("notifications.".$this->type, $params, $language);
    }
}
```
## Model Contents

### Morph Relationship:

The model contains two morph relationships defined using the `morphTo()` method:

- **source()**:
  Establishes a polymorphic relationship with the source model.
  It allows retrieving the `source` model instance associated with the notification.

- **target()**:
  Establishes a polymorphic relationship with the target model.
  It allows retrieving the `target` model instance associated with the notification.

The `morphTo()` method is used to define polymorphic relationships. It enables the model to associate with different models based on the morphable type and ID. In this case, the `source()` and `target()` methods define the morph relationships, allowing the model to dynamically associate with different models based on the context.

By utilizing these morph relationships, you can easily retrieve the corresponding `source` and `target` model instances associated with the notification, enabling you to perform various operations and access related data efficiently.

### create_notification:

```php
public static function create_notification($type, $source, $target, $data)
{
    return Notification::create([
        'type'        => $type,
        'source_type' => $source ? get_class($source) : null,
        'source_id'   => $source ? $source->id : null,
        'target_type' => $target ? get_class($target) : null,
        'target_id'   => $target ? $target->id : null,
        'data'        => json_encode($data),
    ]);
}
```

The `create()` method is used to store the new `Notification` instance in the database. It accepts an array of attributes as its argument, which are assigned values based on the provided parameters.

The `source_type` attribute is set to the class name of the `$source` model using the `get_class()` function. If `$source` is null, the attribute is set to null.

The `source_id` attribute is set to the `id` of the `$source` model if it exists. Otherwise, it is set to null.

The `target_type` attribute is set to the class name of the `$target` model using `get_class()`. If `$target` is null, the attribute is set to null.

The `target_id` attribute is set to the `id` of the `$target` model if it exists. If `$target` is null, it is set to null.

The `data` attribute is set by encoding the `$data` array into JSON format using the `json_encode()` function. This allows additional data to be stored along with the notification.

### send_to_all_users:

The `send_to_all_users` method is used to send a notification to all verified users with the role "user". Here is the code:

```php
public function send_to_all_users()
{
    $users_ids = User::where('role', 'user')->where('verified', true)->pluck('id');
    $data = [];
    foreach ($users_ids as $user_id) {
        $data[$user_id] = ['read' => 0];
    }
    $this->users()->attach($data);
    $this->send($users_ids);
}
```

This method performs the following steps:

1. It retrieves the IDs of all verified users with the role "user" from the `User` model using the `where` method with the specified conditions. The `pluck` method is used to retrieve only the `id` column of the matching users and store them in the `$users_ids` variable.

2. An empty array, `$data`, is initialized to store the data for attaching the notification to users.

3. A `foreach` loop is used to iterate over each user ID in `$users_ids`. Inside the loop, an entry is added to the `$data` array for each user ID, with the key as the user ID and the value as an array with the `'read'` key set to `0`, indicating that the notification is unread for that user.

4. The `attach` method is called on the `users()` relationship of the current object (assuming it has a relationship called `users`). This method attaches the notification to the users specified in the `$data` array, marking it as unread for each user.

5. Finally, the [send](#send-method) method is called, passing the `$users_ids` array as an argument. This method is responsible for sending the notification to the specified users.

By utilizing the `send_to_all_users` method, you can easily send a notification to all verified users with the role "user" in your application, ensuring that it is marked as unread for each user.

### send-method

The `send` function is responsible for sending push notifications to users of a mobile app using Firebase Cloud Messaging (FCM).

```php
private function send($users_ids)
{
    $message = [];
    $registration_ids = [];
    foreach (Language::$all_languages as $lang) {
        $message[$lang] = $this->get_message($lang);
        $registration_ids[$lang] = [];
    }
    
    $fcm_tokens = FCMToken::whereIn('user_id', $users_ids)->get();
    foreach ($fcm_tokens as $fcm_token) {
        $registration_ids[$fcm_token->language][] = $fcm_token->fcm_token;
    }

    $key = env('FIREBASE_KEY', '');

    foreach (Language::$all_languages as $lang) {
        if (count($registration_ids[$lang]) > 0) {
            Http::withHeaders([
                'Content-Type' => 'application/json',
                'Authorization' => "key=$key",
            ])->post('https://fcm.googleapis.com/fcm/send', [
                'registration_ids' => $registration_ids[$lang],
                'notification' => [
                    'title' => 'Ebla Mart',
                    'body' => $message[$lang],
                ]
            ]);
        }
    }
}
```

This method performs the following steps:

1. An empty array, `$message`, is initialized to store the message content for different languages.

2. An empty array, `$registration_ids`, is initialized to store the Firebase Cloud Messaging (FCM) registration IDs for each language.

3. A `foreach` loop is used to iterate over each language in `Language::$all_languages`. Inside the loop, the `get_message($lang)` method is called to retrieve the message content for the corresponding language, and it is stored in the `$message` array. An empty array is also added to the `$registration_ids` array for the current language.

4. The FCM tokens associated with the provided user IDs (`$users_ids`) are retrieved from the `FCMToken` model and stored in the `$fcm_tokens` variable.
   
*FCM tokens are unique identifiers assigned to mobile devices by FCM. These tokens allow the server to send push notifications to specific devices.*
*On the mobile app side, the app needs to request and obtain the FCM token from the Firebase SDK on the device. The FCM token is then sent to the server and associated with the corresponding user. This allows the server to send push notifications to the specific device.*

5. Another `foreach` loop is used to iterate over each `$fcm_token` in `$fcm_tokens`. Inside the loop, the FCM registration ID (`$fcm_token->fcm_token`) is added to the `$registration_ids` array for the corresponding language (`$fcm_token->language`).

6. The Firebase API key is retrieved from the environment configuration using `env('FIREBASE_KEY', '')` and stored in the `$key` variable.

*The `FIREBASE_KEY` key serves as the authorization token for sending requests to the FCM service.*

7. Another `foreach` loop is used to iterate over each language in `Language::$all_languages`. Inside the loop, it checks if there are any registration IDs (`count($registration_ids[$lang]) > 0`) for the current language.

8. If there are registration IDs available, it makes a POST request to the FCM service (`https://fcm.googleapis.com/fcm/send`) using the `Http` client. The request includes the registration IDs and the notification title and body. The request is authenticated by providing the Firebase API key in the `Authorization` header.

### get_message function

```php
public function get_message($language)
{
    $params = [];
    if ($this->type == "account_verification") {
        $params['user_name'] = $this->target->name;
    }

    return __("notifications.".$this->type, $params, $language);
}
```

By calling the `get_message` function and passing the desired language, you can retrieve the translated message for the notification based on its type and any additional parameters.

- It checks if the notification type (`$this->type`) is equal to "account_verification". If it is, it adds a parameter `user_name` to the `$params` array, with the value of the name property of the target object (`$this->target->name`).

- The message is retrieved using Laravel's translation function `__()` with the key `"notifications.".$this->type`, the parameters `$params`, and the specified language.

[🔝 Back to contents](#contents)

### **product-media**

```php
public function add_product_media(Product $product, Request $request)
{
    $request->validate( [
        'type'   => ['required', 'in:image,video,pdf'],
        'title'  => ['string'],
    ]);

    if($request->type=='image')
        $request->validate( [
            'file'   => ['required', 'mimes:png,jpg,jpeg,gif'],
        ]);
    elseif($request->type=='video')
        $request->validate( [
            'file'   => ['required', 'mimes:mp4,ogx,oga,ogv,ogg,webm'],
        ]);
    elseif($request->type=='pdf')
        $request->validate( [
            'file'   => ['required', 'mimes:pdf'],
        ]);

    $file = upload_file($request->file, 'product', 'products_media');

    $media = new MediaFile([
        'type'          => $request->type,
        'file'          => $file,
        'element_type'  => Product::class,
        'element_id'    => $product->id,
        'title'         => $request->title??null,
    ]);

    $media->save();
    return response()->json(new MediaResource($media), 200);
}
```

This method is responsible for adding media files(images, videos, or PDFs) to a specific product.
The method performs file validation based on the value of the type field.
Once the request and file validations pass, the code proceeds to upload the file using a function called [upload_file()](#upload-file) from the helper class.
`MediaFile` instance is created with the data, including the `type`, `file`, `element_type` (set to `Product::class`), `element_id` (set to the ID of the `$product`), and `title` (set to the value of the `title` field if provided, otherwise set to `null`).

[🔝 Back to contents](#contents)

### **upload-file**

`app\helpers.php`

```php
function upload_file($request_file, $prefix, $folder_name)
{
    $extension = $request_file->getClientOriginalExtension();
    $file_to_store = $prefix . '_' . time() . '.' . $extension;
    $request_file->storeAs('public/' . $folder_name, $file_to_store);
    return $folder_name.'/'.$file_to_store;
}
```

The `upload_file` function is a utility function that handles file uploads within the application. It performs the following tasks:

1. Takes a file as input and generates a unique filename for storage.
2. Stores the file in the specified folder.
3. Returns the relative path of the uploaded file.

It takes three parameters:

1. `$request_file`: The file obtained from the request.
2. `$prefix`: A prefix string used to create a unique filename.
3. `$folder_name`: The name of the folder where the file will be stored.

The function returns the relative path of the uploaded file. This path is obtained by concatenating the `$folder_name` and the unique filename generated for the file.

The `upload_file` function is placed within the app helper class, which provides common functions for various tasks. These helper functions are globally accessible throughout the application without needing to import or instantiate any specific class.

### **languages**

`app\Http\Controllers\Dashboard\LanguageController.php`

```php
public function __construct()
{
    $this->middleware('auth:sanctum')->except(['index','show']);
    $this->middleware('role:super_admin|admin')->except(['index','show']);
    $this->middleware('verified')->except(['index','show']);
}
```
.
.
.
```php
public function store(Request $request)
{
    $request->validate( [
        'code'   => ['required', 'string' , 'unique:languages'],
    ]);

    LanguageMiddleware::$all_languages[] = $request->code;

    $request->validate( [
        'title'  => ['required', LanguageMiddleware::rule()],
    ]);

    $language = Language::createWithTranslations([
        'code'  => $request->code,
        'title' => $request->title,
    ]);

    return response()->json(new LanguageResource($language), 200);
}
```
#### Constructor Method

it has three middlewares:

1. `auth:sanctum` middleware: This middleware requires the user to be authenticated using the Sanctum authentication system. It is applied to all methods except `index` and `show`.

2. `role:super_admin|admin` middleware: This middleware ensures that the user has the role of either 'super_admin' or 'admin'. It is applied to all methods except `index` and `show`.

3. `verified` middleware: This middleware checks if the user has a verified email address. It is applied to all methods except `index` and `show`.

#### Store Method

The store method handles the validation of the incoming request and creates a new Language record with translations. It performs additional logic related to translations for associated models. Finally, it returns a JSON response with the created Language resource.

It is invoked by the route defined in the `routes\api.php` file:

```php
Route::apiResource('languages', LanguageController::class);
```

The method takes a `Request` object as a parameter and follows these steps:

1. Validates the `code` field from the request, requiring it to be a unique string.

2. Adds the value of the `code` field from the request to the static property `$all_languages` defined in the [LanguageMiddleware](#language-middleware) class. This keeps track of all the languages.

3. Validates the `title` field from the request using the `LanguageMiddleware::rule()` method.

4. Calls the `createWithTranslations([...])` method on the `Language` model to create a new Language instance with translations based on the provided data.
It uses the `createWithTranslations` method on the [HasTranslation trait class](#has-translation) that depending on language model, passing an array of data containing the 'code' and 'title' fields from the request. This method handles the creation of a new Language record in the database, along with any associated translations.

1. Returns a JSON response containing the newly created Language resource. It uses the [LanguageResource](#language-resource) class to transform the `$language` object into a JSON representation.

### **language-middleware**

`app\Http\Middleware\Language.php`

```php
class Language
{
    public static $all_languages = [];
    public static $language='en';

    public function handle(Request $request, Closure $next)
    {
        $lang=$request->headers->get('accept-language')??'en';
        if(!in_array($lang, Language::$all_languages))
            $lang = 'en';
        Language::$language = $lang;
        return $next($request);
    }

    public static function rule(array $languages=null){
        return 'required_array_keys:'.implode(',',$languages??Language::$all_languages);
    }
}
```
The middleware class that is responsible for extracting and validating the language from the request, setting the currently selected language for the application.

`$lang=$request->headers->get('accept-language')??'en';`
This line retrieves the value of the 'accept-language' header from the incoming request using the $request object. If the header is not present or does not have a value, the default language 'en' is assigned to the $lang variable.

`if(!in_array($lang, Language::$all_languages)) $lang = 'en';`
This conditional statement checks if the language specified in the $lang variable is present in the $all_languages array. If it is not found, the default language 'en' is assigned to $lang.

## public static function rule(array $languages=null)
This static method accepts an optional array of languages as a parameter. Tis method returns a validation rule string that checks if the selected language is one of the specified languages in the *$languages array* using implode method that join the elements of an array into a string and concatenate all the elements of the *$languages array* (or the default $all_languages array) into a string, using ',' as the separator.

### **has-translation**

`app\Traits\HasTranslation.php`

```php
public static function createWithTranslations($attributes = [], Model|null $parent = null, array $languages = null){
    $class = __CLASS__;
    $instance = new $class;

    $translated_columns = $instance->translated_columns??[];
    $translated_attributes = [];

    $fillable = $instance->fillable??[];
    $fillable_attributes = [];

    foreach($attributes as $key => $value)
    {
        if(in_array($key,$translated_columns))
            $translated_attributes[$key] = $value;
        if(in_array($key,$fillable))
            $fillable_attributes[$key] = $value;
    }

    $model = self::create($fillable_attributes, $parent);
    $model->storeTranslations($translated_attributes, $languages);
    $model->loadTranslations();
    return $model;
}
.
.
.
public function storeTranslations($translations, array $languages = null)
{
    foreach($translations as $attribute => $values)
        foreach (($languages??LanguageMiddleware::$all_languages) as $lang)
            $this->translations()->create([
                'language'         => $lang,
                'attribute'        => $attribute,
                'value'            => $values[$lang]??'',
            ]);
}
.
.
.
public function loadTranslations(){
    $trans=$this->translations->where('language',LanguageMiddleware::$language);
    foreach($trans as $t)
    {
        $attr = $t->attribute;
        $this->$attr=$t->value;
    }

    $trans=$this->translations;

    foreach($trans as $t)
    {
        $attr = $t->attribute . '_array';
        if(!$this->$attr)
            $this->$attr=[];
        $a=$this->$attr;
        $a[$t->language]=$t->value;
        $this->$attr=$a;
    }
}
```
The `createWithTranslations` method* I used to create a new instance of a model class with the provided attributes. It separates the attributes into translated and fillable attributes, stores the model with fillable attributes in the database, handles translations, and returns the created model instance.

`'$class = __CLASS__;'`
This line retrieves the fully qualified class name of the current class.

`$model = self::create($fillable_attributes, $parent);`
This line calls the create method of the current class with the *$fillable_attributes* array and $parent as arguments. This method creates a new instance of the model and persists it in the database with the provided attributes.

The code then calls `storeTranslations` function, that loops over the translations and languages, creating new translation records in the associated model's translation table.

After storing the translations, calls the `loadTranslations` method to load the translations into the model. It retrieves translations associated with the model instance and updates the model's attributes based on those translations. This step assumes the existence of a relationship named `translations` and utilizes the `LanguageMiddleware::$language` variable from the [LanguageMiddleware ](#language-middleware) class to filter translations for a specific language.

### **language-resource**

`app\Http\Resources\LanguageResource.php`

```php
class LanguageResource extends JsonResource
{
    public function toArray($request)
    {
        return [
            'id'          => $this->id,
            'code'        => $this->code,
            'title'       => $this->title,    
            'title_array' => $this->title_array??null,
        ];
    }
}
```

[🔝 Back to contents](#contents)
### **offers**

`app\Http\Controllers\api\OfferController.php`

```php
public function __construct()
{
    if (array_key_exists('HTTP_AUTHORIZATION', $_SERVER)) {
        $this->middleware('auth:sanctum')->only(['show', 'index']);
    }
    $this->middleware('auth:sanctum')->except(['index', 'show']);
    $this->middleware('role:super_admin|admin')->except(['index', 'show']);
    $this->middleware('verified')->except(['index', 'show']);
}
```

...

```php
public function show(Offer $offer)
{
    $user = Auth::user();
    $offer->loadTranslations();
    $offer_ratings = Rating::select('element_id', DB::raw('AVG(rate) as avg_rate, COUNT(id) as count_rate'))
                            ->where('element_type', Offer::class)
                            ->groupBy('element_id')->get()->keyBy('element_id');
    
    if (isset($offer_ratings[$offer->id])) {
        $offer->rates_count = $offer_ratings[$offer->id]->count_rate;
        $offer->rate = (float)$offer_ratings[$offer->id]->avg_rate;
    }
    
    if ($user && in_array($user->role, ['super_admin', 'admin'])) {
        return response()->json(new OfferAdminResource($offer), 200);
    }
    
    return response()->json(new OfferResource($offer), 200);
}
```

*Constructor* method:

The constructor method initializes the OfferController class. It defines several middlewares based on specific conditions:

- If the 'HTTP_AUTHORIZATION' key exists in the `$_SERVER` global array, the 'auth:sanctum' middleware is applied to the 'show' and 'index' methods.
- The 'auth:sanctum' middleware is also applied to all methods except 'index' and 'show'.
- The 'role:super_admin|admin' middleware is applied to all methods except 'index' and 'show'.
- The 'verified' middleware is applied to all methods except 'index' and 'show'.

*show* method:

The *show* method retrieves the authenticated user, loads translations for the offer, fetches the ratings for the offer, and updates the offer object with the count and average rate if available. Then, it returns a JSON response based on the user's role.

- If the user is authenticated and has the role of 'super_admin' or 'admin', the method returns a JSON response using the 'OfferAdminResource' class to format the offer data.
- For other authenticated users or unauthenticated users, the method returns a JSON response using the 'OfferResource' class to format the offer data.

The code also executes a query on the Rating model using the query builder. It selects the 'element_id' column and calculates the average rate (AVG) and count of ratings (COUNT) for each element.

[🔝 Back to contents](#contents)

### **add-to-favorites**

`app\Http\Controllers\api\FavoriteController.php`

```php
public function store(Request $request)
{
    $request->validate([
        'product_id' => ['required', 'exists:products,id'],
    ]);

    $product = Product::find($request->product_id);
    $user = to_user(Auth::user());
    $exists = $user->favorites()->where('product_id', $request->product_id)->exists();

    if (!$exists) {
        $user->favorites()->attach($request->product_id);
    }

    $product->is_favorite = true;

    $products_ratings = Rating::select('element_id', DB::raw('AVG(rate) as avg_rate, COUNT(id) as count_rate'))
                                ->where('element_type', Product::class)
                                ->groupBy('element_id')->get()->keyBy('element_id');

    if (isset($products_ratings[$product->id])) {
        $product->rates_count = $products_ratings[$product->id]->count_rate;
        $product->rate = (float)$products_ratings[$product->id]->avg_rate;
    }

    return response()->json(new ProductResource($product), 200);
}
```

**Check if Product Exists in User's Favorites:**

The code checks if the authenticated user has already added the product to their favorites. It does this by querying the user's favorites relationship defined in the User model and filtering by the 'product_id'.

The `exists` method is called on the query, which returns a boolean indicating whether a record exists.

If the product does not exist in the user's favorites, the code uses the `attach` method on the user's favorites relationship to add the product to the user's favorites. This establishes a many-to-many relationship between the user and the product.

The code executes a query on the Rating model using the *query builder*. It then checks if the product's ID exists in the `$products_ratings` collection using `isset($products_ratings[$product->id])`. If it does exist, it assigns the count of ratings to `$product->rates_count` and the average rate (converted to a float) to `$product->rate`. This updates the `$product` object with the count of ratings and average rate, retrieved from the `$products_ratings` collection.

[🔝 Back to contents](#contents)

### **search**

`app\Http\Controllers\api\SearchController.php`

```php
public function search(Request $request)
{
    $request->validate([
        'q'                => ['string'],
        'categories_ids'   => ['array'],
        'categories_ids.*' => ['exists:categories,id'],
        'brands_ids'       => ['array'],
        'brands_ids.*'     => ['exists:brands,id'],
        'min_price'        => ['numeric', 'min:0'],
        'max_price'        => ['numeric', 'min:0'],
    ]);

    $favorites = [];
    $user = to_user(Auth::user());

    if($user)
        $favorites = $user->favorites()->pluck('product_id')->toArray();

    $filtered_products_ids = Translation::where('translation_type', Product::class)
                                        ->where('value', 'LIKE', '%'.$request->q.'%')
                                        ->pluck('translation_id');

    $filtered_offers_ids = Translation::where('translation_type', Offer::class)
                                        ->where('value', 'LIKE', '%'.$request->q.'%')
                                        ->pluck('translation_id');
                                        
    $p = Product::with('media','ratings')->where('available', true)
                                ->whereIn('id', $filtered_products_ids)
                                ->where('quantity', '>', 0);
                                
    $o = Offer::with('media','ratings')->where('expire_at', '>=', Carbon::now())->whereIn('id', $filtered_offers_ids);

    $products_ratings = Rating::select('element_id', DB::raw('AVG(rate) as avg_rate,COUNT(id) as count_rate'))
                                ->where('element_type', Product::class)
                                ->groupBy('element_id')->get()->keyBy('element_id');

    $offers_ratings = Rating::select('element_id', DB::raw('AVG(rate) as avg_rate,COUNT(id) as count_rate'))
                                ->where('element_type', Offer::class)
                                ->groupBy('element_id')->get()->keyBy('element_id');

    if($request->categories_ids)
        $p->whereIn('category_id', $request->categories_ids);

    if($request->brands_ids)
        $p->whereIn('brand_id', $request->brands_ids);

    if($request->min_price){
        $p->where('retail_price','>=' ,$request->min_price);
        $o->where('retail_price','>=' ,$request->min_price);
    }
    if($request->max_price){
        $p->where('retail_price','<=' ,$request->max_price);
        $o->where('retail_price','<=' ,$request->max_price);
    }

    $products = $p->get();
    $offers = $o->get();

    if($request->categories_ids){
        $_offers = $offers;
        $offers = [];
        foreach($_offers as $offer){
            foreach($offer->products as $product){
                if(in_array($product->category_id, $request->categories_ids))
                {
                    $offers[] = $offer;
                    break;
                }
            }
        }
    }
    
    if($request->brands_ids){
        $_offers = $offers;
        $offers = [];
        foreach($_offers as $offer){
            foreach($offer->products as $product){
                if(in_array($product->brand_id, $request->brands_ids))
                {
                    $offers[] = $offer;
                    break;
                }
            }
        }
    }

    foreach($products as $product)
    {
        $product->is_favorite = in_array($product->id, $favorites);
        if(isset($products_ratings[$product->id])){
            $product->rates_count = $products_ratings[$product->id]->count_rate;
            $product->rate = (float)$products_ratings[$product->id]->avg_rate;
        }
    }
    
    foreach($offers as $offer)
    {
        if(isset($offers_ratings[$offer->id])){
            $offer->rates_count = $offers_ratings[$offer->id]->count_rate;
            $offer->rate = (float)$offers_ratings[$offer->id]->avg_rate;
        }
    }

    return [
        'products' => ProductResource::collection($products),
        'offers'   => OfferResource::collection($offers),
        ];
}
```
This method describes a search operation based on various criteria, including the search string, category IDs, brand IDs, and price range. It retrieves and filters products and offers accordingly, while also fetching and calculating ratings for the retrieved items.

The code starts by validating the incoming request using the `validate` method. It checks the following parameters of the request:

- `q`: A string parameter used for searching.
- `categories_ids`: An array parameter containing category IDs.
- `categories_ids.*`: Each element of `categories_ids` should exist as an ID in the `categories` table.

If the user is authenticated, the code retrieves the user's favorite products by plucking the `product_id` values from the user's favorites relationship and converts them to an array.

### Filter Products and Offers by Translation

The code queries the Translation model to filter products and offers based on a search string. For the product query (`$p`), it performs the following steps:

1. Includes only products that are marked as available (`where('available', true)`).
2. Filters products based on the `$filtered_products_ids` obtained from the translations query.
3. Includes only products with a positive quantity (`where('quantity', '>', 0)`).

### Additional Filtering

The method applies additional filters based on the request parameters: (`categories_ids`, `brands_ids`, `min_price`, `max_price`).

### Ratings Calculation

The code retrieves and calculates ratings for the products and offers using the Rating model. It calculates the average rate and count of ratings for each element (product or offer) and stores them in separate collections: `$products_ratings` and `$offers_ratings`. The ratings are grouped by the element ID using the `groupBy` method.

The method returns an array containing the filtered `products` and `offers` as collections. These collections can be further formatted and transformed as needed using the appropriate classes before being returned as a JSON response.

[🔝 Back to contents](#contents)

## Relationship Between Offer and Product (offer-product)

The relationship between Offer and Product models is a many-to-many relationship defined by the `belongsToMany` method. The intermediate table `offer_product` is used to establish the relationship and includes additional data such as the quantity of each product associated with an offer.

### Migration File: create_offer_product_table.php

The migration file `create_offer_product_table.php` defines the schema for the intermediate table `offer_product`. It includes the following columns:

```php
Schema::create('offer_product', function (Blueprint $table) {
    $table->id();
    $table->foreignId('offer_id')->constrained('offers');
    $table->foreignId('product_id')->constrained('products');
    $table->integer('quantity');
});
```

### Offer Model: Offer.php

In the Offer model, the relationship is defined by passing `Product::class` as the related model and specifying the intermediate table name as `'offer_product'` using the `withPivot` method. The `withPivot('quantity')` method indicates that the intermediate table should include the `'quantity'` column.

```php
class Offer extends Model
{
    // ...

    public function products(){
        return $this->belongsToMany(Product::class, 'offer_product')->withPivot('quantity');
    }

    // ...
}
```

### OfferProduct Model: OfferProduct.php

The OfferProduct model represents the intermediate table between the Offer and Product models in the many-to-many relationship. It includes the following properties and methods:

- The `$timestamps` property is set to `false` to disable timestamp columns in the intermediate table.
- The `$table` property is set to `'offer_product'`, specifying the name of the intermediate table.
- The `$fillable` property defines the columns that can be mass-assigned when creating or updating an OfferProduct model instance.

```php
class OfferProduct extends Model
{
    use HasFactory;

    public $timestamps = false;
    protected $table = 'offer_product';

    protected $fillable = [
        'offer_id',
        'product_id',
        'quantity',
    ];
}
```

This setup allows you to establish a many-to-many relationship between Offer and Product models, with the intermediate table `offer_product` storing additional data such as the quantity of each product associated with an offer.


[🔝 Back to contents](#contents)