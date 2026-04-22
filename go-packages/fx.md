
# Uber FX: গোছানো নোট (বাংলায়)

## ১. ম্যানুয়াল DI বনাম FX

### সমস্যা: ম্যানুয়াল ডিপেন্ডেন্সি ইনজেকশন

ম্যানুয়াল পদ্ধতিতে আপনাকে সঠিক ক্রমে সব ডিপেন্ডেন্সি তৈরি করতে হয়। প্রজেক্ট বড় হলে এটা ঝামেলাপূর্ণ ও ভুলপ্রবণ হয়ে ওঠে।

```go
// Manual wiring: ক্রম ঠিক রাখা জরুরি
db := NewDB()
repo := NewRepo(db)
service := NewService(repo)
```

### সমাধান: FX-এর ডিক্লারেটিভ DI

FX-এ আপনি শুধু **কীভাবে তৈরি হবে** তা বলে দেন (Constructor)। ফ্রেমওয়ার্ক নিজেই গ্রাফ সমাধান করে নেয়, আপনি যে অর্ডারেই লিখুন না কেন।

```go
fx.New(
    fx.Provide(NewDB),      // যে কোনো জায়গায় লেখা যাবে
    fx.Provide(NewRepo),
    fx.Provide(NewService),
)
```

---

## ২. তিনটি মূল ধারণা (শুরুর পয়েন্ট)

FX মূলত তিনটি স্তম্ভের ওপর দাঁড়িয়ে আছে।

| ধারণা | কাজ | উপমা |
| :--- | :--- | :--- |
| **`fx.Provide`** | **নির্মাণ:** কোনো অবজেক্ট *কীভাবে* বানাতে হবে তা FX-কে জানানো। আপনি Constructor ফাংশন সরবরাহ করেন। | ব্লুপ্রিন্ট জমা দেওয়া। |
| **`fx.Invoke`** | **কার্যকর:** অ্যাপ্লিকেশন শুরুর সময় কোন ফাংশন *রান* করাতে হবে তা জানানো। | "main" ফাংশনের ট্রিগার। |
| **`fx.Lifecycle`** | **নিয়ন্ত্রণ:** অ্যাপ্লিকেশন চালু হওয়া এবং বন্ধ হওয়ার সময়ের হুক ম্যানেজ করা। | লাইফসাইকেল ব্যবস্থাপনা। |

### উদাহরণ

```go
func NewServer(lc fx.Lifecycle) *Server {
    srv := &Server{}
    lc.Append(fx.Hook{
        OnStart: func(ctx context.Context) error {
            fmt.Println("সার্ভার চালু হচ্ছে...")
            go srv.Listen()
            return nil
        },
        OnStop: func(ctx context.Context) error {
            fmt.Println("সার্ভার বন্ধ হচ্ছে...")
            return srv.Shutdown(ctx)
        },
    })
    return srv
}

func main() {
    fx.New(
        fx.Provide(NewServer),
        fx.Invoke(func(*Server) {
            // এটি কনস্ট্রাকশনের পর রান হয়। সাধারণত খালি থাকে।
            // কিন্তু সার্ভার ইতিমধ্যে Lifecycle হুকের মাধ্যমে চলছে!
            fmt.Println("অ্যাপ শুরু হয়ে গেছে।")
        }),
    ).Run()
}
```

---

## ৩. `fx.Annotate`: একই টাইপের একাধিক ইনস্ট্যান্স ব্যবস্থাপনা

### সমস্যা
FX টাইপ অনুযায়ী কাজ করে। আপনার যদি দুটি কনস্ট্রাক্টর থাকে যারা উভয়ই `*sql.DB` রিটার্ন করে (যেমন: Primary ও Replica), FX ত্রুটি দেখাবে: `cannot provide function ... (*sql.DB) already provided`.

### সমাধান: Result Tags ও Param Tags

`fx.Annotate` ব্যবহার করে **নামযুক্ত ট্যাগ** যোগ করে আলাদা করতে হবে।

**ধাপ ১: কনস্ট্রাক্টরে ট্যাগ লাগানো (`ResultTags`)**

```go
// database/db.go
func NewPrimaryDB(cfg *config.Config) *Database {
    fmt.Println("প্রাইমারি ডিবি সংযোগ হচ্ছে:", cfg.DBHost)
    return &Database{Host: cfg.DBHost}
}

func NewReplicaDB(cfg *config.Config) *Database {
    fmt.Println("রেপ্লিকা ডিবি সংযোগ হচ্ছে:", cfg.DBHost)
    return &Database{Host: "replica-" + cfg.DBHost}
}

// main.go
fx.Provide(
    fx.Annotate(database.NewPrimaryDB, fx.ResultTags(`name:"primary"`)),
    fx.Annotate(database.NewReplicaDB, fx.ResultTags(`name:"replica"`)),
)
```

**ধাপ ২: নির্দিষ্ট ট্যাগ চাওয়া (`ParamTags`)**

যখন কোনো কনস্ট্রাক্টরের নির্দিষ্ট ইনস্ট্যান্স দরকার হয়, তখন `ParamTags` ব্যবহার করুন। **মূল নিয়ম:** `ParamTags`-এ **প্রতিটি** প্যারামিটারের জন্য একটি এন্ট্রি দিতে হবে। যেসব প্যারামিটারে ট্যাগ লাগবে না সেখানে খালি স্ট্রিং `""` বসাতে হবে।

```go
// server/server.go
// প্যারামিটার ক্রম: Lifecycle, Config, PrimaryDB, ReplicaDB
func NewServer(lc fx.Lifecycle, cfg *config.Config, primary *Database, replica *Database) *Server {
    // ... primary ও replica ব্যবহার করুন
}

// main.go
fx.Provide(
    fx.Annotate(server.NewServer, fx.ParamTags(
        ``,                // fx.Lifecycle এর জন্য (কোনো ট্যাগ নেই)
        ``,                // *config.Config এর জন্য (কোনো ট্যাগ নেই)
        `name:"primary"`,  // *Database যার ট্যাগ primary
        `name:"replica"`,  // *Database যার ট্যাগ replica
    )),
)
```

---

## ৪. `fx.As`: কংক্রিট টাইপকে ইন্টারফেস হিসেবে উপস্থাপন

### সমস্যা
আপনি FX-এর মাধ্যমে একটি **ইন্টারফেস** (যেমন `io.Writer`) ইনজেক্ট করতে চান, কিন্তু আপনি সরবরাহ করছেন একটি **কংক্রিট স্ট্রাক্ট** (যেমন `*MyLogger`)। ডিফল্টভাবে FX শুধু কংক্রিট টাইপই রেজিস্টার করে।

### সমাধান
`fx.As` ব্যবহার করে FX-কে বলুন: "যখন কেউ `I` ইন্টারফেস চাইবে, তুমি এই কংক্রিট ইমপ্লিমেন্টেশন দিও।"

```go
type Writer interface {
    Write(p []byte) (n int, err error)
}

type MyLogger struct{}

func NewLogger() *MyLogger {
    return &MyLogger{}
}

func (l *MyLogger) Write(p []byte) (n int, err error) {
    return os.Stdout.Write(p)
}

func main() {
    fx.New(
        fx.Provide(
            fx.Annotate(
                NewLogger,
                fx.As(new(Writer)), // জাদু: *MyLogger কে Writer ইন্টারফেস হিসেবে রেজিস্টার করল
            ),
        ),
        fx.Invoke(func(w Writer) {
            // এটি কাজ করবে! w আসলে *MyLogger
            w.Write([]byte("ইন্টারফেস থেকে হ্যালো\n"))
        }),
    ).Run()
}
```

---

## ৫. Value Groups: ডিপেন্ডেন্সির তালিকা একত্রিত করা

### সমস্যা
আপনি একটি ইন্টারফেসের **সকল** ইমপ্লিমেন্টেশন (যেমন: সব `Plugin` বা `Middleware`) একটি স্লাইসে জমা করতে চান, কিন্তু ম্যানুয়ালি তালিকা আপডেট করতে চান না।

### সমাধান
`fx.Group` ব্যবহার করুন। সরবরাহকারীরা গ্রুপে যোগ করবে এবং ভোক্তা সেই স্লাইসটি গ্রহণ করবে।

**ধাপ ১: গ্রুপে সরবরাহ করা (`ResultTags`)**

```go
func NewPluginA() Plugin {
    return &APlugin{}
}

func NewPluginB() Plugin {
    return &BPlugin{}
}

fx.Provide(
    fx.Annotate(NewPluginA, fx.ResultTags(`group:"plugins"`)),
    fx.Annotate(NewPluginB, fx.ResultTags(`group:"plugins"`)),
)
```

**ধাপ ২: গ্রুপ গ্রহণ করা (`ParamTags` বা `fx.In`)**

```go
type Params struct {
    fx.In

    PluginList []Plugin `group:"plugins"` // FX স্বয়ংক্রিয়ভাবে A ও B এখানে যোগ করবে
}

func NewPluginManager(p Params) *Manager {
    fmt.Printf("লোড হয়েছে %d টি প্লাগইন\n", len(p.PluginList))
    // PluginList = [*APlugin, *BPlugin]
    return &Manager{plugins: p.PluginList}
}

fx.Provide(NewPluginManager)
```

**পরামর্শ: `fx.In` দিয়ে সহজীকরণ**
লম্বা `ParamTags` ব্যবহার না করে `fx.In` স্ট্রাকট দিয়ে গ্রুপ নেওয়া যায় পরিষ্কারভাবে:

```go
type ManagerDeps struct {
    fx.In

    Log    *Logger
    Cf     *Config
    Plugs  []Plugin `group:"plugins"`
}

func NewManager(deps ManagerDeps) *Manager {
    // ...
}
```


## More explanation 
### ২. Annotate কী?

**কাজ:** একটি ফাংশনের গায়ে বাড়তি নির্দেশনা (টীকা) লাগানো।

**গঠন:**
```go
fx.Annotate(
    তোমার ফাংশন,
    fx.As(...),        // টীকা ১
    fx.ResultTags(...), // টীকা ২
    // ... আরও টীকা
)
```

---

### ৩. As কী?

**কাজ:** একটি কংক্রিট টাইপকে (struct) ইন্টারফেস হিসেবে রেজিস্টার করা।

**গঠন:**
```go
fx.As(new(ইন্টারফেসের_নাম))
```

**উদাহরণ:**
```go
fx.Annotate(
    NewAuthService,
    fx.As(new(auth_ports.AuthService)),
)
```

---

### ৪. কখন As লাগে আর কখন লাগে না

#### পরিস্থিতি ১: As লাগে না

```go
// ফাংশন নিজেই ইন্টারফেস রিটার্ন করছে
func NewAuthService() auth_ports.AuthService {
    return &AuthService{}
}

// Fx-তে সরাসরি ব্যবহার
fx.Provide(NewAuthService)  // ✅ As লাগবে না
```

#### পরিস্থিতি ২: As লাগে

```go
// ফাংশন কংক্রিট struct রিটার্ন করছে
func NewAuthService() *AuthService {
    return &AuthService{}
}

// Fx-তে ইন্টারফেস হিসেবে রেজিস্টার করতে As লাগবে
fx.Provide(
    fx.Annotate(
        NewAuthService,
        fx.As(new(auth_ports.AuthService)),  // ✅ এটা লাগবে
    ),
)
```

#### পরিস্থিতি ৩: একাধিক ইন্টারফেসের জন্য As

```go
type Reader interface { Read() }
type Writer interface { Write() }

type File struct{}

func NewFile() *File {
    return &File{}
}

fx.Provide(
    fx.Annotate(
        NewFile,
        fx.As(new(Reader)),  // File কে Reader হিসেবেও চিনবে
        fx.As(new(Writer)),  // File কে Writer হিসেবেও চিনবে
    ),
)
```

---

### ৫. ResultTags কী?

**কাজ:** রিটার্ন করা জিনিসের গায়ে নামের তকমা লাগানো।

**গঠন:**
```go
fx.ResultTags(`name:"তোমার_নাম"`)
```

**উদাহরণ:**
```go
fx.Annotate(
    NewAuthService,
    fx.ResultTags(`name:"primary_auth"`),
)

// পরে অন্য জায়গায় ব্যবহার
func NewController(authSvc auth_ports.AuthService) *Controller {
    return &Controller{authSvc: authSvc}
}

// Fx-তে
fx.Provide(
    fx.Annotate(
        NewController,
        fx.ParamTags(`name:"primary_auth"`),  // নির্দিষ্ট নামের জিনিস চাওয়া
    ),
)
```

---

### ৬. ParamTags কী?

**কাজ:** ফাংশনের ইনপুট প্যারামিটার কোন তকমার জিনিস নেবে তা ঠিক করা।

**গঠন:**
```go
fx.ParamTags(`name:"তোমার_নাম"`)
```

**উদাহরণ:**
```go
// দুইটা আলাদা ডাটাবেজ কানেকশন
fx.Provide(
    fx.Annotate(NewUserDB, fx.ResultTags(`name:"user_db"`)),
    fx.Annotate(NewOrderDB, fx.ResultTags(`name:"order_db"`)),
)

// নির্দিষ্ট ডাটাবেজ চাওয়া
fx.Provide(
    fx.Annotate(
        NewUserService,
        fx.ParamTags(`name:"user_db"`),  // user_db নামের কানেকশন নেবে
    ),
)
```

---

### ৭. পুরো উদাহরণ একসাথে

```go
package main

import "go.uber.org/fx"

// ========== ইন্টারফেস ==========
type AuthService interface {
    Login() string
}

// ========== বাস্তবায়ন ==========
type authService struct{}

func (a *authService) Login() string {
    return "logged in"
}

// ========== কনস্ট্রাক্টর ==========
func NewAuthService() *authService {
    return &authService{}
}

func NewController(svc AuthService) *Controller {
    return &Controller{svc: svc}
}

// ========== Controller ==========
type Controller struct {
    svc AuthService
}

// ========== Fx মডিউল ==========
var Module = fx.Options(
    fx.Provide(
        fx.Annotate(
            NewAuthService,                     // আসল ফাংশন (*authService রিটার্ন করে)
            fx.As(new(AuthService)),            // Interface হিসেবে রেজিস্টার
            fx.ResultTags(`name:"main_auth"`),  // নামের তকমা
        ),
    ),
    fx.Provide(
        fx.Annotate(
            NewController,
            fx.ParamTags(`name:"main_auth"`),   // নির্দিষ্ট তকমার জিনিস নেবে
        ),
    ),
)

func main() {
    fx.New(Module).Run()
}
```

---

### ৮. মনে রাখার সহজ সূত্র

| টার্ম | কী নেয় | কী করে |
|:---|:---|:---|
| `fx.Annotate` | ফাংশন + টীকাসমূহ | ফাংশনে টীকা লাগায় |
| `fx.As` | `new(Interface)` | Struct → Interface রূপান্তর |
| `fx.ResultTags` | `` `name:"x"` `` | রিটার্নে তকমা লাগায় |
| `fx.ParamTags` | `` `name:"x"` `` | ইনপুটে নির্দিষ্ট তকমার জিনিস চায় |

---

### ৯. কখন কী লাগবে - চেকলিস্ট

```go
// ✅ লাগবে না
func New() MyInterface { return &myStruct{} }
fx.Provide(New)

// ✅ লাগবে
func New() *myStruct { return &myStruct{} }
fx.Provide(fx.Annotate(New, fx.As(new(MyInterface))))

// ✅ একাধিক ইন্টারফেস
func New() *myStruct { return &myStruct{} }
fx.Provide(fx.Annotate(New, 
    fx.As(new(InterfaceA)), 
    fx.As(new(InterfaceB)),
))

// ✅ নামের তকমা লাগবে
func New() *myStruct { return &myStruct{} }
fx.Provide(fx.Annotate(New, fx.ResultTags(`name:"primary"`)))

// ✅ নির্দিষ্ট নামের জিনিস চাই
func NewController(svc MyInterface) *Controller { return &Controller{svc} }
fx.Provide(fx.Annotate(NewController, fx.ParamTags(`name:"primary"`)))
```
