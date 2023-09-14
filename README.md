A date as a string is less reliable than an object instance, e.g. a Carbon-instance. It's recommended to pass Carbon objects between classes instead of date strings. Rendering should be done in the display layer (templates):
A date as a string is less reliable than an object instance, e.g. a Carbon-instance. It's recommended to pass Carbon objects between classes instead of date strings. Rendering should be done in the display layer (templates):
## Contents

[Authentication](#authentication)

[Store new language functionality](#languages)

[Change settings function](#change-settings)

[Store service function](#store-subscriber)

[Export to excel function function](#excel)


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

### **languages**

app\Http\Controllers\Dashboard\LanguageController.php:

Constructor method, it has three middlewares:

First middleware is set to 'auth:sanctum', which means that the user must be authenticated using the Sanctum authentication system to access the methods of this class. However, the except(['index','show']) part specifies that the middleware should not be applied to the index and show methods.

Second middleware is set to 'role:super_admin|admin', which indicates that the user must have the role of either 'super_admin' or 'admin' to access the methods of this class.

Third middleware, the 'verified' middleware typically comes with Laravel's built-in email verification feature. It ensures that the user must have a verified email address to access the methods of this class.

```php
    public function __construct()
    {
        $this->middleware('auth:sanctum')->except(['index','show']);
        $this->middleware('role:super_admin|admin')->except(['index','show']);
        $this->middleware('verified')->except(['index','show']);
    }
```
This method handles the validation of the incoming request, creates a new Language record with translations, performs some additional logic related to translations for associated models, and returns a JSON response with the created Language resource. 

LanguageMiddleware::$all_languages[] = $request->code;
This line adds the value of the code field from the request to the static property $all_languages defined in the [LanguageMiddleware ](#language-middleware) class. It appends the new language code to the array, which keeps track of all the languages.

createWithTranslations([...]) method,
This line creates a new Language instance with translations based on the provided data. It uses the createWithTranslations method on the [HasTranslation trait class](#has-translation) tha depending on language model, passing an array of data containing the 'code' and 'title' fields from the request. This method handles the creation of a new Language record in the database, along with any associated translations.

<!-- LanguageController::add_all_models_translations_for($language->code);
This line calls the add_all_models_translations_for method on the LanguageController class, passing the code of the newly created language as an argument. The purpose of this method is not apparent from the provided code snippet, but it likely performs some additional logic related to translations for all models associated with the language. -->

This method at the end returns a JSON response containing the newly created Language resource. It uses the [LanguageResource class](#language-resource) to transform the $language object into a JSON representation.

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

        LanguageController::add_all_models_translations_for($language->code);

        return response()->json(new LanguageResource($language), 200);
    }
```
.
.
.
app\Http\Middleware\Language.php:

The middleware class that is responsible for extracting and validating the language from the request, setting the currently selected language for the application.

public function handle(Request $request, Closure $next)
This method is the main entry point for the middleware and is responsible for handling incoming requests.
$lang=$request->headers->get('accept-language')??'en';
This line retrieves the value of the 'accept-language' header from the incoming request using the $request object. If the header is not present or does not have a value, the default language 'en' is assigned to the $lang variable.

if(!in_array($lang, Language::$all_languages)) $lang = 'en';
This conditional statement checks if the language specified in the $lang variable is present in the $all_languages array. If it is not found, the default language 'en' is assigned to $lang.

public static function rule(array $languages=null)
This static method accepts an optional array of languages as a parameter. Tis method returns a validation rule string that checks if the selected language is one of the specified languages in the $languages array using implode method that join the elements of an array into a string and concatenate all the elements of the $languages array (or the default $all_languages array) into a string, using ',' as the separator.

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
app\Traits\HasTranslation.php:

createWithTranslations method is used to create a new instance of a model class with the provided attributes. It separates the attributes into translated and fillable attributes, stores the model with fillable attributes in the database, handles translations, and returns the created model instance.

'$class = __CLASS__;'
This line retrieves the fully qualified class name (including the namespace) of the current class.

$model = self::create($fillable_attributes, $parent);
This line calls the create method of the current class with the $fillable_attributes array and $parent as arguments. This method creates a new instance of the model and persists it in the database with the provided attributes.

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
        $model->makeHidden(['translations']);
        return $model;
    }
```
.
.
.
```php
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
```
.
.
.
```php
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

[🔝 Back to contents](#contents)

### **change-settings**

app\Http\Controllers\FormController.php:

```php
    public function change_settings(Request $request)
    {
        $request->validate([
            'volunteer_email'   => ['nullable','email'],
            'risk_email'        => ['nullable','email'],
            'join_email'        => ['nullable','email'],
            'enabled_volunteer' => ['nullable','in:0,1'],
            'enabled_risk'      => ['nullable','in:0,1'],
            'enabled_join'      => ['nullable','in:0,1'],
        ]);

        $this->setEnv('APP_VOLUNTEER_ENABLE',$request->enabled_volunteer);
        $this->setEnv('APP_AT_RISK_ENABLE',$request->enabled_risk);
        $this->setEnv('APP_JOIN_ENABLE',$request->enabled_join);

        $this->setEnv('APP_VOLUNTEER_EMAIL',$request->volunteer_email);
        $this->setEnv('APP_AT_RISK_EMAIL',$request->risk_email);
        $this->setEnv('APP_JOIN_EMAIL',$request->join_email);

        return back()->withStatus(__('Settings Successfully Changed'));
    }
```

```php
    function setEnv($name, $value)
    {
        $path = base_path('.env');
        if (file_exists($path)) {
            file_put_contents($path, str_replace(
                $name . '=' . env($name) , $name . '=' . $value, str_replace(
                $name . '="' . env($name) .'"', $name . '="' . $value.'"', file_get_contents($path)
            )));
        }
    }
```

[🔝 Back to contents](#contents)

### **store-subscriber**

app\Http\Controllers\SubscribeController.php:

```php
    public function store(Request $request)
    {
        $request->validate([
            'email' => 'required|email',
        ]);

        $token = Str::random(32);

        $subscribe = Subscription::firstOrCreate(['email' => $request->email],[
            'email' => $request->email,
            'token' => $token,
        ]);

        if ($subscribe->wasRecentlyCreated)
            return back()->withStatus(__('User have successfully subscribed!'));

        return back()->withInput()->with('error', 'User are already subscribed!');
    }
```

[🔝 Back to contents](#contents)

### **excel**

```php
    public function exportToExcel($type)
    {
        $fileName = 'applications' . time() . Auth::user()->id . '.xlsx';
        $forms = Form::where('type', $type)->get();

        if($type == 'join_ind')
            return Excel::download(new JoinIndExport($forms), $fileName);
        elseif($type == 'join_org')
            return Excel::download(new JoinOrgExport($forms), $fileName);
        elseif($type == 'risk')
            return Excel::download(new AtRiskExport($forms), $fileName);
        elseif($type == 'volunteer')
            return Excel::download(new VolunteerExport($forms), $fileName);

    }
```

[🔝 Back to contents](#contents)