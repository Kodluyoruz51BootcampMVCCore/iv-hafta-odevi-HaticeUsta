In this blogpost I want to show you difference between the ASP.NET Core methods `AddMvc()` and `AddMvcCore()` when working with ASP.NET Core.

[ASPNETCore-WebAPI-Sample](https://github.com/FabianGosebrink/ASPNETCore-WebAPI-Sample)

### Startup.cs

When creating an ASP.NET Core WebAPI you often see a Startup.cs file to configure your services and configure your pipeline. Thats what the methods `ConfigureServices(IServiceCollection services)` and `Configure(IApplicationBuilder app, IHostingEnvironment env, ILoggerFactory loggerFactory)` are for.

`ConfigureServices` is preparing your services for being used as you configure them.

> Here is also the place to add dependency injection, but that is another seperate topic

You can have a lot of configuration in here. But we want to focus on the main point: Adding the mvc framework.

When starting with “File” –> “New Project” in Visual Studio the default setting in the method is `AddMvc()`. And it works straight away. Lets take a look:

```csharp
public void ConfigureServices(IServiceCollection services)
{
    // ...
    services.AddMvc();
}
```

When now an API with Controllers kicks in we can consume it like normal.

```
dotnet run` makes the API work on `localhost:5000
GET localhost:5000/api/house
```

brings

```javascript
[
  {
    id: 1,
    street: 'Street1',
    city: 'Town1',
    zipCode: 1234
  },
  {
    id: 2,
    street: 'Street2',
    city: 'Town2',
    zipCode: 1234
  },
  {
    id: 3,
    street: 'Street3',
    city: 'Town3',
    zipCode: 1234
  },
  {
    id: 4,
    street: 'Street4',
    city: 'Town4',
    zipCode: 1234
  }
];
```

What happens, if we change `AddMvc()` to `AddMvcCore()`?

```csharp
public void ConfigureServices(IServiceCollection services)
{
    // ...
    services.AddMvcCore();
}
```

Lets run the same command again:

```
GET localhost:5000/api/house
```

now brings a `406` Error saying “Not Acceptable”.

If we check the console which hosts the webapi we see more information about this error:

> warn: Microsoft.AspNetCore.Mvc.Internal.ObjectResultExecutor[1]No output formatter was found for content type '’ to write the response.

So we deactivated something we better should have not deactivated ;-).

The error says that we do not have an output formatter which can parse our output. Even if we add an accept header in the request like `Accept: application/json` we would get the same error message, because the server does not know how ot handle the respose.

Lets take a closer look on what is the real difference between `AddMvc()` and `AddMvcCore()`.

Due to the fact that the framework is open source we can take a look at the sources:

```csharp
public static IMvcBuilder AddMvc(this IServiceCollection services)
{
    if (services == null)
    {
        throw new ArgumentNullException(nameof(services));
    }

    var builder = services.AddMvcCore();

    builder.AddApiExplorer();
    builder.AddAuthorization();

    AddDefaultFrameworkParts(builder.PartManager);

    // Order added affects options setup order

    // Default framework order
    builder.AddFormatterMappings();
    builder.AddViews();
    builder.AddRazorViewEngine();
    builder.AddCacheTagHelper();

    // +1 order
    builder.AddDataAnnotations(); // +1 order

    // +10 order
    builder.AddJsonFormatters();

    builder.AddCors();

    return new MvcBuilder(builder.Services, builder.PartManager);
}
```

From [MvcServiceCollectionExtensions.cs](https://github.com/aspnet/Mvc/blob/dev/src/Microsoft.AspNetCore.Mvc/MvcServiceCollectionExtensions.cs#L25-L56) tells us, that we are adding the complete MVC Services you need to get the whole MVC functionality.

It is adding Authorization, the RazorViewEngine and the JsonFormatters we need to get our output going. And most interesting it is also calling the `AddMvcCore()` method itself.

So if we use the `AddMvc()` method we got the ability to render view with razor and so on.

Lets have a look at `AddMvcCore()` on the other hand:

[MvcCoreServiceCollectionExtensions.cs](https://github.com/aspnet/Mvc/blob/48546dbb28ee762014f49caf052dc9c8a01eec3a/src/Microsoft.AspNetCore.Mvc.Core/DependencyInjection/MvcCoreServiceCollectionExtensions.cs#L37-L54)

```csharp
public static IMvcCoreBuilder AddMvcCore(this IServiceCollection services)
{
    if (services == null)
    {
        throw new ArgumentNullException(nameof(services));
    }

    var partManager = GetApplicationPartManager(services);
    services.TryAddSingleton(partManager);

    ConfigureDefaultFeatureProviders(partManager);
    ConfigureDefaultServices(services);
    AddMvcCoreServices(services);

    var builder = new MvcCoreBuilder(services, partManager);

    return builder;
}
```

This method is a lot shorter and only adding the basic things to get started. But both methods are returning an IMvcCoreBuilder. Interesting is the `AddMvcCoreServices(services);` method which is adding the ability to return FileContents, RedirectToRouteResults, ActionResolvers, use Controllers, use routing and so on. Really basic functionality.

So when using `AddMvcCore()` we have to add everything by ourselves. This means, that we only have in our application what we really want and for example do not include the razor functionality which we do not need anyway.

Now that we know the difference between those two methods: How can we get our webapi going again? We still have the error and we can not return any data.

We can fix that by simply telling ASP.NET that it should use JsonFormatters like

```csharp
public void ConfigureServices(IServiceCollection services)
{
	// ...

	// Add framework services.
	services.AddMvcCore().AddJsonFormatters();
}
```

If we now call our

```
GET localhost:5000/api/house
```

again we should see the output as json like we expected it to be.

I hope this clarified a bit what is the main difference between AddMvc() and AddMvcCore().

**Data Annotations Nedir?
**

[MVC](https://www.mshowto.org/tag/mvc) uygulamasında veri tabanı tablolarını Code First yöntemi ile oluşturmaya başladığımızda yapılan validasyon işlemlerine Data Annotations denir.

![img](https://www.mshowto.org/images/articles/2019/01/012719_1714_MVCDataAnno1.png)

**Resim-1
**

Visual Studio geliştirme ortamını açalım, File>New>Project yolunu izleyerek MVC projemizi oluşturmaya başlayalım.

![img](https://www.mshowto.org/images/articles/2019/01/012719_1714_MVCDataAnno2.png)

**Resim-2
**

Templates>Visual C#>Web şablonlarından ASP.NET Web Application seçeneğini seçelim, projemize Name kısmından isim verelim. OK tuşuna tıklayarak devam edelim.

![img](https://www.mshowto.org/images/articles/2019/01/012719_1714_MVCDataAnno3.png)

**Resim-3
**

Empty şablonunu seçtikten sonra MVC seçeneğini de seçelim ve OK tuşuna tıklayıp projemizi oluşturalım.

![img](https://www.mshowto.org/images/articles/2019/01/012719_1714_MVCDataAnno4.png)

**Resim-4
**

Controllers seçeneğine sağ tıklayalım Add>Controller seçeneği ile Controller oluşturalım.

![img](https://www.mshowto.org/images/articles/2019/01/012719_1714_MVCDataAnno5.png)

**Resim-5
**

MVC 5 Controller – Empty seçeneğini seçelim ve Add butonunu tıklayarak devam edelim.

![img](https://www.mshowto.org/images/articles/2019/01/012719_1714_MVCDataAnno6.png)

**Resim-6
**

Controller dosyamıza bir isim verelim ve Add seçeneği ile ekleme işlemini gerçekleştirelim.

![img](https://www.mshowto.org/images/articles/2019/01/012719_1714_MVCDataAnno7.png)

**Resim-7
**

Views klasörüne sağ tıklayalım Add>View seçeneğini seçelim.

![img](https://www.mshowto.org/images/articles/2019/01/012719_1714_MVCDataAnno8.png)

**Resim-8
**

View adımızı _Layout olarak verelim ve sayfalarımız için bir adet Master Page oluşturalım.

![img](https://www.mshowto.org/images/articles/2019/01/012719_1714_MVCDataAnno9.png)

**Resim-9
**

ActionResult metodumuza sağ tıklayalım Add View seçeneği ile projemize View ekleyelim.

![img](https://www.mshowto.org/images/articles/2019/01/012719_1714_MVCDataAnno10.png)

**Resim-10
**

Index adı metot adı ile aynı olarak gelmektedir. Add seçeneğine tıklayalım ve Layout dosyamızın yolunu belirterek View ekleme işlemini gerçekleştirelim.

![img](https://www.mshowto.org/images/articles/2019/01/012719_1714_MVCDataAnno11.png)

**Resim-11
**

Models klasörüne sağ tıklayalım Add>Class seçeneğini seçelim.

![img](https://www.mshowto.org/images/articles/2019/01/012719_1714_MVCDataAnno12.png)

**Resim-12
**

Class dosyamıza isim verip Add seçeneği ile ekleme işlemini gerçekleştirelim.

![img](https://www.mshowto.org/images/articles/2019/01/012719_1714_MVCDataAnno13.png)

**Resim-13
**

Class dosyamızda iki adet değişken ataması yapalım.

![img](https://www.mshowto.org/images/articles/2019/01/012719_1714_MVCDataAnno14.png)

**Resim-14
**

Tools seçeneğini seçelim NuGet Package Manager >Manage NuGet Packages For Solution seçeneğini seçelim.

![img](https://www.mshowto.org/images/articles/2019/01/012719_1714_MVCDataAnno15.png)

**Resim-15
**

Browse seçeneğine Validation yazalım ve Unobtrusive Validation seçeneğini seçerek Install edelim.

![img](https://www.mshowto.org/images/articles/2019/01/012719_1714_MVCDataAnno16.png)

**Resim-16
**

Yüklenecek olan Javascript paketlerini görmekteyiz OK seçeneğine tıklayalım ve yükleme işlemini gerçekleştirelim.

![img](https://www.mshowto.org/images/articles/2019/01/012719_1714_MVCDataAnno17.png)

**Resim-17
**

Lisans anlaşmasını kabul ederek devam edelim.

![img](https://www.mshowto.org/images/articles/2019/01/012719_1714_MVCDataAnno18.png)

**Resim-18
**

Yüklenen dosyalar Scripts klasörünün altında yer alacaktır.

![img](https://www.mshowto.org/images/articles/2019/01/012719_1714_MVCDataAnno19.png)

**Resim-19
**

Layout dosyamıza NuGet ile yüklediğimiz Jquery dosyalarımızı referans olarak ekleyelim.

![img](https://www.mshowto.org/images/articles/2019/01/012719_1714_MVCDataAnno20.png)

**Resim-20
**

Web Config dosyamıza Connection String nesnemizi ekleyelim.

![img](https://www.mshowto.org/images/articles/2019/01/012719_1714_MVCDataAnno21.png)

**Resim-21
**

Models klasörüne sağ tıklayalım Add>Class seçeneğini seçelim.

![img](https://www.mshowto.org/images/articles/2019/01/012719_1714_MVCDataAnno22.png)

**Resim-22
**

[Context](https://www.mshowto.org/tag/context) yani veri tabanı bağlantımızı sağlayacak olan Class eklememizi yapalım.

![img](https://www.mshowto.org/images/articles/2019/01/012719_1714_MVCDataAnno23.png)

**Resim-23
**

Tools seçeneğini seçelim NuGet Package Manager >Manage NuGet Packages For Solution seçeneğini seçelim.

![img](https://www.mshowto.org/images/articles/2019/01/012719_1714_MVCDataAnno24.png)

**Resim-24
**

Entity Framework seçeneğini seçelim ve sağ tarafta projemizi seçerek Install seçeneğini seçelim. Veri tabanı işlemleri için bu paketi yüklememiz gerekmektedir.

![img](https://www.mshowto.org/images/articles/2019/01/012719_1714_MVCDataAnno25.png)

**Resim-25
**

Context dosyamıza gelerek veri tabanı bağlantımızı yapacağımız sınıfımızın tanımlarını gerçekleştirelim.

![img](https://www.mshowto.org/images/articles/2019/01/012719_1714_MVCDataAnno26.png)

**Resim-26
**

Models klasörüne sağ tıklayalım Add>Class seçeneğini seçelim.

![img](https://www.mshowto.org/images/articles/2019/01/012719_1714_MVCDataAnno27.png)

**Resim-27
**

Veri tabanı tablomuzu ve validasyon işlemlerimizi yapacağımız Class eklememizi gerçekleştirelim.

![img](https://www.mshowto.org/images/articles/2019/01/012719_1714_MVCDataAnno28.png)

**Resim-28
**

Required nesnelerimiz ile zorunlu alanlarımızı ve yanlarında mesajlarımızı da yazalım.

![img](https://www.mshowto.org/images/articles/2019/01/012719_1714_MVCDataAnno29.png)

**Resim-29
**

Controller dosyamıza View içerisinden göndereceğimiz nesnelerin boş olup olmadığını kontrol ederek geriye uyarı mesajımızı ve boş bir model nesnesi gönderelim. Kullandığımız ModelState nesnesi geçerliliği kontrol edip geriye mesaj göndermemize yarar.

![img](https://www.mshowto.org/images/articles/2019/01/012719_1714_MVCDataAnno30.png)

**Resim-30
**

Index View dosyamızın içerisinde nesnelerimizi oluşturalım nesne isimlendirmelerimiz veri tabanı Class dosyası içerisinde olanlar ile aynı olacaktır. Programımızı çalıştıralım.

![img](https://www.mshowto.org/images/articles/2019/01/012719_1714_MVCDataAnno31.png)

**Resim-31
**

Projemizde hiç veri girmeden formu gönderme işlemini gerçekleştirelim.

![img](https://www.mshowto.org/images/articles/2019/01/012719_1714_MVCDataAnno32.png)

**Resim-32
**

Form kontrolleri çalışarak uyarı mesajlarımız ekrana gelmiştir.

Data Annotations işlemlerini görmüş olduk.





# [Snapshot Nedir?](https://www.tayfundeger.com/snapshot-nedir.html)



**Snapshot nedir?** **Snapshot nasıl çalışır?** sorularına cevap vereceğim bu yazımda. Aslında sanallaştırma ortamını kullanan birçok kişi snapshot’ı aktif olarak kullanıyor. Ben bu yazımda Snapshot kavramını biraz daha detaylı olarak anlatacağım çünkü hala snapshot’ı backup gibi kullanılıyor.

Basit olarak açıklamak gerekirse, snapshot’ı kısa süreli çalışmalar öncesinde backup almak yerine snapshot’ı kullanabiliriz. **Snapshot** sayesinde sanal makinamızın anlık ekran görüntüsü alınır ve siz herhangi bir sebepten dolayı sanal makina üzerinde yaptığınız işlemleri geri almak istediğinizde almış olduğunuz snapshot’a revert diyerek eski haline geri dönüş yapabilirsiniz.

Snapshot’ı sanal makinanız power off, suspend veya power on durumdayken alabilirsiniz. Snapshot alırken karşınıza çıkan seçenekler ile ilgili aşağıdaki yazımı okuyabilirsiniz.

[Sanal makina üzerindeki snapshot kavramları](https://www.tayfundeger.com/sanal-makina-uzerindeki-snapshot-kavramlari.html)

[![img](https://www.tayfundeger.com/wp-content/uploads/VMware-Reklam2.png)](https://www.tayfundeger.com/iletisim)

[![snapshot-work](https://www.tayfundeger.com/wp-content/uploads/snapshot-work-1-582x403.png)](https://www.tayfundeger.com/wp-content/uploads/snapshot-work-1.png)

Snapshot eğer kontrollü bir şekilde alınır ve takip edilirse herhangi bir sorun oluşturmaz. Snapshot alındıktan sonra değişen data’lar disk üzerinde yer kapladığı için datastore’unuzu doldurabilir. Ayrıca snapshot bir chain yapısından oluştuğu için, birden fazla snapshot alınması durumunda eğer aradaki chain’lerden birtanesi silinir veya bir sebepden dolayı corrupt duruma düşerse komple sanal makinayı kaybetmeniz kuvvet ile muhtemeldir. Yani snapshot’ın aslında faydası olduğu gibi eğer takibi yapılmaz ise size zararıda olur.

Snapshot kısa süreli işlemler için kullanılması tavsiye ediliyor. Örneğin bir update yapacaksınız ve update 1 veya 2 hafta sürecek. Bu durumda snapshot almak çok doğru değil. Böyle bir operasyonda backup veya clone alarak daha rahat birşekilde işlemlerinize devam edebilirsiniz. VMware’in yayınlamış olduğu Snapshot Best Practices’lerde bir snapshot’ın 24-72 saat arasında silinmesi gerektiği belirtiliyor. Eğer snapshot’ı bu süreden uzun tutarsanız hem virtual machine’in konrolunde hemde performans anlamında problemler yaşayabilirsiniz. Çünkü bir snapshot aldığınızda yeni bir vmdk dosyası oluşmaz. Değişen dataların tutulduğu bir delta vmdk dosyası oluşur. Dolayısıyla siz bir veriyi yazmak veya okumak istediğinizde bu hem vmdk üzeirnde hemde delta vmdk üzerinde işlem yapacaktır. Bu durumda da performans sorunları mutlaka ortaya çıkacaktır. Snapshot’ıda silmek istediğinizde yukarıda belirtmiş olduğum delta vmdk dosyası ile ana vmdk dosyanız birleştirilir. Eğer snapshot’ınız büyük ise veya 24-72 saatten uzun bir snapshot ise muhtemelen bu snapshot’ı silmenizde vakit alacaktır. Örneğin bir mail sunucunuz var ve üzerinde snapshot aldınız 1 hafta sonrada bunu silmek istediniz. Silme işlemini başlattıkdan sonra işlem uzun sürecektir. Bununda sebebi Mail Server üzerinde sürekli değişen dataların olmasıdır. Tabi bu süre kullanmış olduğunu disk’e görede değişkenlik gösterecektir. Yine VMware snapshot best practices’lerine göre bir sanal makine üzerinde 2 veya 3 snapshot’dan fazla bulundurmamanızı önerir. Ancak maximum desteklenen snapshot sayısı 32’dir.

Özellikle database, mail server gibi sanal makinelerde uzun süreli snapshot bekletmemeye dikkat edin. Bu sunucular üzeirndeki datalar değiştiği için sizin datastore’unuzuda doldurabilir. Datastore’unuzda da yer kalmadığında bütün sanal makineleriniz down duruma geçicektir. Bunun önüne geçmek için vCenter’ınıza provision space alarm’ı tanımlayabilirsiniz.





How do I add a calendar in HTML using jQuery?

To **create** the **calendar** we only need to **add** a div **tag** with an id. Then before the body closing **tag** we need to **add** the **jQuery** and the **jQuery** UI script. We also need to call the “datepicker”, so you need to use the same id that you've used on the div.

How can add event in jQuery calendar?

**How to use it:**

1. Include jQuery library and the jQuery e-calendar's CSS and Javascript in your web page. < link rel = "stylesheet" href = "css/jquery.e-calendar.css" > ...
2. Create an empty element that will be served as a calendar container. ...
3. Initialize the plugin with options and add your own events using JS array object.

How do I use Datepicker?

The **datepicker** is tied to a standard form input field. Focus on the input (click, or **use** the tab key) to open an interactive calendar in a small overlay. Choose a date, click elsewhere on the page (blur the input), or hit the Esc key to close. If a date is chosen, feedback is shown as the input's value.

#### First- FirstOrDefault ve Single- SingleOrDefault

There is

- a semantical difference
- a performance difference

between the two.

**Semantical Difference:**

- `FirstOrDefault` returns a first item of potentially multiple (or default if none exists).
- `SingleOrDefault` assumes that there is a single item and returns it (or default if none exists). Multiple items are a violation of contract, an exception is thrown.

**Performance Difference**

- `FirstOrDefault` is usually faster, it iterates until it finds the element and only has to iterate the whole enumerable when it doesn't find it. In many cases, there is a high probability to find an item.
- `SingleOrDefault` needs to check if there is only one element and therefore always iterates the whole enumerable. To be precise, it iterates until it finds a second element and throws an exception. But in most cases, there is no second element.

**Conclusion**

- Use `FirstOrDefault` if you don't care how many items there are **or** when you can't afford checking uniqueness (e.g. in a very large collection). When you check uniqueness on adding the items to the collection, it might be too expensive to check it again when searching for those items.
- Use `SingleOrDefault` if you don't have to care about performance too much and want to make sure that the assumption of a single item is clear to the reader and checked at runtime.

In practice, you use `First` / `FirstOrDefault` often even in cases when you assume a single item, to improve performance. You should still remember that `Single` / `SingleOrDefault` can improve readability (because it states the assumption of a single item) and stability (because it checks it) and use it appropriately.



##### Null Check

```java
if (acct != null && !acct.isEmpty())
```

Note the use of `&&` here, rather than your `||` in the previous code; also note how in your previous code, your conditions were wrong anyway - even with `&&` you would only have entered the `if` body if `acct` *was* an empty string.

```java
if (!Strings.isNullOrEmpty(acct))
```



# ASP.NET Razor Pages vs MVC: How Do Razor Pages Fit in Your Toolbox?

MATT WATSONAUGUST 16, 2017[DEVELOPER TIPS, TRICKS & RESOURCES](https://stackify.com/developers/)

As part of the release of [.NET Core 2.0](https://stackify.com/net-core-2-0-changes/), there are also some updates to ASP.NET. One of those is the addition of a new web framework for creating a “page” without the full complexity of ASP.NET MVC. New Razor Pages are a slimmer version of the MVC framework and in some ways an evolution of the old “.aspx” WebForms.

In this article, we are going to cover some of the finer points of using ASP.NET Razor Pages vs MVC.

- The basics of Razor Pages
- ASP.NET MVVM vs MVC
- Pros and cons on Razor Pages
- Using Multiple GET or POST Actions via Handlers
- Why you should use Razor Pages for everything
- Code comparison of ASP.NET Razor Page vs MVC

### The Basics: What are ASP.NET Razor Pages?

A [Razor Page](https://docs.microsoft.com/en-us/aspnet/core/mvc/razor-pages/) is very similar to the view component that ASP.NET MVC developers are used to. It has all the same syntax and functionality.

The key difference is that the model and controller code is also included within the Razor Page itself. It is more an MVVM (Model-View-ViewModel) framework. It enables two-way data binding and a simpler development experience with isolated concerns.

Here is a basic example of a Razor Page using inline code within a @functions block. It is actually recommended to put the PageModel code in a separate file. This is more akin to how we did code behind files with ASP.NET WebForms.

```
@page
@model IndexModel
@using Microsoft.AspNetCore.Mvc.RazorPages

@functions {
    public class IndexModel : PageModel
    {
        public string Message { get; private set; } = "In page model: ";

        public void OnGet()
        {
            Message += $" Server seconds  { DateTime.Now.Second.ToString() }";
        }
    }
}

<h2>In page sample</h2>
<p>
    @Model.Message
</p>
```

### We Have Two Choices Now: ASP.NET MVVM or MVC

You could say that we now have the choice of an MVC or MVVM framework. I’m not going to go into all the details of MVC vs MVVM. This [article](http://www.dotnettricks.com/learn/designpatterns/understanding-mvc-mvp-and-mvvm-design-patterns) does a good job of that with some examples. MVVM frameworks are most noted for two-way data binding of the data model.

MVC works well with apps that have a lot of dynamic server views, single page apps, REST APIs, and [AJAX calls](https://stackify.com/return-ajax-response-asynchronous-javascript-call/). Razor Pages are perfect for simple pages that are read-only or do basic data input.

MVC has been all the rage recently for web applications across most programming languages. It definitely has its pros and cons. ASP.NET WebForms was designed as an MVVM solution. You could argue that Razor Pages are an evolution of the old WebForms.

------





I’ve been doing ASP.NET development for about 15 years. I am very well versed in all the ASP.NET frameworks. Based on my playing around with the new Razor Pages, these are my pros and cons and how I would see using them.

#### Pro: More organized and less magical

I don’t know about you, but the first time I ever used ASP.NET MVC I spent a lot of time trying to figure out how it worked. The naming of things and the dynamically created routes caused a lot of magic that I wasn’t used to. The fact that /Home/ goes to HomeController.Index() that loads a view file from “Views\Home\Index.cshtml” is a lot of magic to get comfortable with when starting out.

Razor Pages don’t have any of that “magic” and the files are more organized. You have a Razor View and a code behind file just like WebForms did. Versus with MVC having separate files in different directories for the controller, view, and model.

Compare simple MVC and Razor Page projects. (Will show more code differences later in this article.)

![Compare MVC vs Razor Page Files](https://stackify.com/wp-content/uploads/2017/08/img_5994b41da1992.png)

COMPARE MVC VS RAZOR PAGE FILES

#### Pro: Single Responsibility

If you have ever used an MVC framework before, you have likely seen some huge controller classes that are filled with many different actions. They are like a virus that grows over time as things get added.

With Razor Pages, each page is self-contained with its view and code organized together. This follows the [Single Responsibility Principle](https://en.wikipedia.org/wiki/Single_responsibility_principle).

 

### Using Multiple GET or POST Actions via Handlers

By default a Razor Page is designed to have a single OnGetAsync and OnPostAsync method. If you want to have different actions within your single page you need to use what is called a **handler**. You would need this if your page has AJAX call backs, multiple possible form submissions, or other scenarios.

So for example, if you were using a Kendo grid and wanted the grid to load via an AJAX call, you would need to use a handler to handle that AJAX call back. Any type of single page application would use a lot of handlers or you should point all of those AJAX calls to an MVC controller.

I made an additional method called OnGetHelloWorldAsync() in my page. How do I call it?

From my research there seems to be **3 different ways to use handlers**:

1. Querystring  – Example: “/managepage/2177/?handler=helloworld”
2. Define as a route in your view: @page “{handler?}” and then use /helloworld in the url
3. Define on your input submit button in your view. Example: <input type=”submit” asp-page-handler=”JoinList” value=”Join” />

Can learn more about multiple page handlers [here](https://docs.microsoft.com/en-us/aspnet/core/mvc/razor-pages/#multiple-handlers-per-page).

*Special thanks to those who left comments about this! Article updated!*

 

### Why you should use Razor Pages for everything! (maybe?)

I could make an argument that Razor Pages are the perfect solution to anything that is essentially a web page within your app. It would draw a clear line in the sand that any HTML “pages” in your app are true pages. Currently, an MVC action could return an HTML view, JSON, a file, or anything. Using Pages would **force a separation** between how you load the page and what services the AJAX callbacks.

Think about it, this solves a lot of problems with this forced separation.

| **Razor Page**HTML Views | **MVC/Web API**REST API calls, SOA |
| ------------------------ | ---------------------------------- |
|                          |                                    |

This would prevent MVC controllers that contains tons of actions that are a mix of not only different “pages” in your app but also a mixture of AJAX callbacks and other functions.

Of course, I haven’t actually implemented this strategy yet. It could be terrible or brilliant. Only time will tell how the community ends up using Razor Pages.

### Code Comparison of ASP.NET Razor Page vs MVC

As part of playing around with Razor Pages, I built a really simple form in both MVC and as a Razor Page. Let’s take a look at how the code compares. It is just a text box with a submit button.

Here is my MVC view:

```
@model RazorPageTest.Models.PageClass

<form asp-action="ManagePage">
    <div class="form-horizontal">
        <h4>Client</h4>
        <hr />
        <div asp-validation-summary="ModelOnly" class="text-danger"></div>
        <input type="hidden" asp-for="PageDataID" />
        <div class="form-group">
            <label asp-for="Title" class="col-md-2 control-label"></label>
            <div class="col-md-10">
                <input asp-for="Title" class="form-control" />
                <span asp-validation-for="Title" class="text-danger"></span>
            </div>
        </div>
      
        <div class="form-group">
            <div class="col-md-offset-2 col-md-10">
                <input type="submit" value="Save" class="btn btn-default" />
            </div>
        </div>
    </div>
</form>
```

Here is my MVC controller. (My model is PageClass which just has two properties and is really simple.)

```
   public class HomeController : Controller
    {
        public IConfiguration Configuration;

        public HomeController(IConfiguration config)
        {
            Configuration = config;
        }

        public async Task<IActionResult> ManagePage(int id)
        {
            PageClass page;

            using (var conn = new SqlConnection(Configuration.GetConnectionString("contentdb")))
            {
                await conn.OpenAsync();

                var pages = await conn.QueryAsync<PageClass>("select * FROM PageData Where PageDataID = @p1", new { p1 = id });

                page = pages.FirstOrDefault();
            }

            return View(page);
        }

        [HttpPost]
        [ValidateAntiForgeryToken]
        public async Task<IActionResult> ManagePage(int id, PageClass page)
        {

            if (ModelState.IsValid)
            {
                try
                {
                    //Save to the database
                    using (var conn = new SqlConnection(Configuration.GetConnectionString("contentdb")))
                    {
                        await conn.OpenAsync();
                        await conn.ExecuteAsync("UPDATE PageData SET Title = @Title WHERE PageDataID = @PageDataID", new { page.PageDataID, page.Title});
                    }
                }
                catch (Exception)
                {
                   //log it
                }
                return RedirectToAction("Index", "Home");
            }
            return View(page);
        }
    }
```

**Now let’s compare that to my Razor Page.**

My Razor Page:

```
@page "{id:int}"
@model RazorPageTest2.Pages.ManagePageModel

<form asp-action="ManagePage">
    <div class="form-horizontal">
        <h4>Manage Page</h4>
        <hr />
        <div asp-validation-summary="ModelOnly" class="text-danger"></div>
        <input type="hidden" asp-for="PageDataID" />
        <div class="form-group">
            <label asp-for="Title" class="col-md-2 control-label"></label>
            <div class="col-md-10">
                <input asp-for="Title" class="form-control" />
                <span asp-validation-for="Title" class="text-danger"></span>
            </div>
        </div>

        <div class="form-group">
            <div class="col-md-offset-2 col-md-10">
                <input type="submit" value="Save" class="btn btn-default" />
            </div>
        </div>
    </div>
</form>
```

Here is my Razor PageModel, aka code behind:

```
   public class ManagePageModel : PageModel
    {
        public IConfiguration Configuration;

        public ManagePageModel(IConfiguration config)
        {
            Configuration = config;
        }

        [BindProperty]
        public int PageDataID { get; set; }
        [BindProperty]
        public string Title { get; set; } 

        public async Task<IActionResult> OnGetAsync(int id)
        {
            using (var conn = new SqlConnection(Configuration.GetConnectionString("contentdb")))
            {
                await conn.OpenAsync();
                var pages = await conn.QueryAsync("select * FROM PageData Where PageDataID = @p1", new { p1 = id });

                var page = pages.FirstOrDefault();

                this.Title = page.Title;
                this.PageDataID = page.PageDataID;
            }

            return Page();
        }

        public async Task<IActionResult> OnPostAsync(int id)
        {

            if (ModelState.IsValid)
            {
                try
                {
                    //Save to the database
                    using (var conn = new SqlConnection(Configuration.GetConnectionString("contentdb")))
                    {
                        await conn.OpenAsync();
                        await conn.ExecuteAsync("UPDATE PageData SET Title = @Title WHERE PageDataID = @PageDataID", new { PageDataID, Title });
                    }
                }
                catch (Exception)
                {
                   //log it
                }
                return RedirectToPage("/");
            }
            return Page();
        }
    }
```

#### Deciphering the comparison

The code between the two is nearly identical. Here are the key differences:

- The MVC view part of the code is exactly the same except the Razor Page has “@page” in it.
- ManagePageModel has OnGetAsync and OnPostAsync which replaced the two MVC controller “ManagePage” actions.
- ManagePageModel includes my two properties that were in the separate PageClass before.

In MVC for an HTTP POST, you pass in your object to the MVC action (i.e. “ManagePage(int id, PageClass page)”). With a Razor Page, you are instead using two-way data binding. To get Razor Pages to work correctly with two-way data binding I had to annotate my two properties (PageDataID, Title) with [BindProperty]. My OnPostAsync method only has a single input of the id since the other properties are automatically bound.







































