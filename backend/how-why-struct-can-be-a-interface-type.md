# Go Language: Interface এবং Polymorphism সহজ উদাহরণসহ

এই ডকুমেন্টটিতে Go (গো) ল্যাংগুয়েজের **Interface** এবং **Polymorphism** কনসেপ্টটি একটি বাস্তব উদাহরণের মাধ্যমে সহজভাবে ব্যাখ্যা করা হয়েছে।

## ১. সম্পূর্ণ কোড (The Code)

```go
package main

import (
	"fmt"
)

type Location struct {
	Name string
	Lat  float32
	Lng  float32
}

func (l Location) String() string {
	return fmt.Sprintf("%s (%.4f, %.4f)", l.Name, l.Lat, l.Lng)
}

type RouteResult struct {
	Mode        string
	Checkpoints []Location
	DistanceKm  float32
	DurationMin float32
}

func (r RouteResult) String() string {
	return fmt.Sprintf(
		"[%s] %d checkpoint(s) | distance: %.1f km | time: %.0f min",
		r.Mode, len(r.Checkpoints), r.DistanceKm, r.DurationMin,
	)
}

type RouteStrategy interface {
	BuildRoute(start Location, end Location) RouteResult
}

type CarRouteStrategy struct{}

func (c CarRouteStrategy) BuildRoute(start Location, end Location) RouteResult {
	distanceKm := 12.9
	speedKmph := 40.4
	durationMin := (distanceKm / speedKmph) * 60

	return RouteResult{
		Mode: "car",
		Checkpoints: []Location{
			start,
			end,
		},
		DistanceKm: float32(distanceKm),
		DurationMin: float32(durationMin),
	}
}

func main() {
	home := Location{
		Name: "Home",
		Lat:  32.324,
		Lng:  90.234,
	}

	airport := Location{
		Name: "airport",
		Lat:  31.324,
		Lng:  40.234,
	}

	// আমাদের আলোচনার মূল লাইন
	var strategy RouteStrategy = CarRouteStrategy{}
	result := strategy.BuildRoute(home, airport)
	fmt.Println(result)
}
```

## ২. আমাদের মূল আলোচনার বিষয় (The Confusing Line)

কোডের নিচের এই লাইনটি নতুনদের কাছে প্রায়ই কনফিউজিং লাগে:
```go
var strategy RouteStrategy = CarRouteStrategy{}
```

এটাকে আমরা তিনটি ভাগে ভাগ করতে পারি:

1. **`var strategy`**: আপনি একটা খালি বক্স বা ভেরিয়েবল তৈরি করলেন, যার নাম দিলেন `strategy`।
2. **`RouteStrategy`**: এই বক্সটার একটা বিশেষ নিয়ম বা শর্ত আছে। এই বক্সে আপনি শুধু তাকেই রাখতে পারবেন, যার `BuildRoute` (রাস্তা তৈরি করার) ক্ষমতা আছে। কোডিংয়ের ভাষায় এই নিয়মকে বলে **Interface** (ইন্টারফেস)।
3. **`= CarRouteStrategy{}`**: এবার আপনি সেই বক্সের ভেতর `CarRouteStrategy` নামের একটা জিনিস (Struct) ঢুকিয়ে দিলেন। `{}` দিয়ে বোঝানো হয়েছে আপনি জিনিসটার একটা নতুন কপি তৈরি করে বক্সে রাখছেন।

---

## ৩. বাস্তব উদাহরণ (Real-World Example)

বিষয়টা বোঝার জন্য একটি বাস্তব উদাহরণ চিন্তা করা যাক:

* **`RouteStrategy` (Interface)**: এটি হলো একটি **"ড্রাইভার"** এর চাকরির পদ। পদের শর্ত হলো: প্রার্থীকে অবশ্যই গাড়ি চালাতে (`BuildRoute`) জানতে হবে।
* **`CarRouteStrategy` (Struct)**: এটি হলো **"রহিম"** নামের একজন মানুষ।

আপনার কোডে `func (c CarRouteStrategy) BuildRoute...` লিখে আপনি প্রমাণ করেছেন যে, রহিমের গাড়ি চালানোর ক্ষমতা আছে। অর্থাৎ, রহিম ওই ড্রাইভার পদের শর্ত পূরণ করে ফেলেছে।

এখন ওই লাইনটার (`var strategy RouteStrategy = CarRouteStrategy{}`) মানে হলো:
> *"আমি একটা ড্রাইভারের পদ (`var strategy RouteStrategy`) তৈরি করলাম, এবং সেই পদে রহিমকে (`CarRouteStrategy{}`) বসিয়ে দিলাম।"*

যেহেতু রহিম গাড়ি চালাতে জানে, তাই সে ওই পদের যোগ্য! Go (গো) ল্যাংগুয়েজ তাই এই লাইনটা দেখে কোনো এরর (Error) দেয়নি।

---

## ৪. এভাবে লেখার আসল জাদু বা সুবিধা কী? (The Magic of Interfaces) 

ধরুন, আজকে আপনি গাড়ির জন্য রাস্তা বের করছেন (`CarRouteStrategy`)। কিন্তু কালকে যদি আপনি বাইকের জন্য রাস্তা বের করার একটা নতুন সিস্টেম আনেন, যার নাম `BikeRouteStrategy`? বাইকও তো `BuildRoute` (রাস্তা বানানো) জানে!

তখন আপনি খুব সহজেই শুধু এই একটা লাইন পরিবর্তন করে দিতে পারবেন:
```go
var strategy RouteStrategy = BikeRouteStrategy{}
```

বাকি পুরো কোডে (`strategy.BuildRoute(home, airport)`) আপনাকে আর একদম হাত দিতে হবে না! কারণ `strategy` বক্সটা জানে যে ওর ভেতরে এমন কেউ আছে যে রাস্তা বানাতে পারে, সেটা গাড়িই হোক আর বাইকই হোক। কোড একদম ফ্লেক্সিবল বা পরিবর্তনযোগ্য (Extensible) হয়ে গেল।

```
