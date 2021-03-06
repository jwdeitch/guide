# Templater Engine
The Spiral Templater engine is the default view processor and provides very simple and powerful ways to manage your website templates using html tags. Such way provides very flexible mechanism to combine multiple teams on projects and gives more power into hands on frontend/html developers to create layouts and pages.

> Spiral Templater is not intended to create or replace basic control structures in your templates, if you wish to replace php as you primary templating language - you can create custom view processor and define your own language (using Twig, Smarty and etc).

This specific section of documentation is mainly focused on template inheritance in your projects, if you wish to read about "virtual tags" and the ability to include one template into another read [Extended Templater usage] (expert.md). Make sure that you have already read about [using views] (/components/views.md) in your code.

## Template Inheritance
Before we start using Templater engine let's imagine the classical situation when we have two very similar pages on your website, for example it can be the about-us and contact form. 

About Us:

```html
<!DOCTYPE html>
<html>
<head>
    <title>
        About Us
    </title>
    <link rel="stylesheet" href="/resources/styles/website.css"/>
</head>
<body>
    <div class="header">
       This is our header.
    </div>
    <div class="wrapper">
         Something about us, <?=$someVariable?>. <a href="/some-url/">Some Address</a>.
    </div>
    <div class="footer">
        This is our footer.
    </div>
</body>
</html>
```

Contact form:

```html
<!DOCTYPE html>
<html>
<head>
    <title>
        Contact Us
    </title>
    <link rel="stylesheet" href="/resources/styles/website.css"/>
</head>
<body>
    <div class="header">
       This is our header.
    </div>
    <div class="wrapper">
        <form action="/contact/submit" method="POST">
            <input type="text" name="name">
            ...
        </form>
    </div>
    <div class="footer">
        This is our footer.
    </div>
</body>
</html>
```

As you can notice, both templates use a very similar layout which differs only in it's title and page content. If we wish to simplify page creation we can try to move the header and footer sections of our layouts into separate views and modify our code like so:

```php
<?=$this->container->views->render('header',['title'=>'About Us'])?>
    Something about us.
<?=$this->container->views->render('footer')?>
```

However, this technique has many disadvantages, for example we can easly leave some tag unclosed and your application must spend resources rendering both the footer and header every time your page is loaded. To prevent such problems we can create one universal template for our pages and extend it in our views, let's do that and place the view file into "application/views/layouts/basic.php":

```html
<!DOCTYPE html>
<html>
<head>
    <title>
        <yield:title/>
    </title>
    <link rel="stylesheet" href="/resources/styles/website.css"/>
</head>
<body>
    <div class="header">
       This is our header.
    </div>
    <div class="wrapper">
        <yield:content/>
    </div>
    <div class="footer">
        This is our footer.
    </div>
</body>
</html>
```

As you may notice we kept all of our universal html code in one file and defined two placeholders for title and content blocks. Now we can edit our about us template and utilize the abilities of the Templater engine:

```html
<extends:layouts.basic title="About Us"/>

<block:content>
    Something about us. <a href="/some-url/">Some Address</a>.
</block:content>
```

> One of the nice side effects of using spiral Templater is that any IDE or editor will treat its tags as normal html/xml tags, as a result your content will be nicely indented and mistakes highlighted. 

The resulting html will look exactly the same as before, however our view is now much smaller in size. Let's try to walk through our template to understand what we just did.
First of all we declared that our view file must extend the parent template "layouts.basic". Because we are allowed to use path seperators in html tags - "." will be replaced with "/", which will allow the templater to extend our desired "layouts/basic" view (the current view namespace [default] will be used).

Next, in the extends tag we declared a tag attribute "title", this definition tells the parent layout to replace the "title" block with the given value. These block definitions can be used for simple html strings and php code.

Secondly, we created an html (xml) tag named "block:content" for the same purpose as for title. Because we want to specify content with inner html tags (in our case a) we can't use short, attribute like, definitions.

> Templater tokenizes html code before processing it, this means that you can not use `<` and `>` symbols in any of your tag attribute as it will be invalid html. We can always use longer blocks definition like in case of "<block:content>". We can demonstrate that both definitions are identical by rewriting our template:

```html
<extends:layouts.basic/>

<block:title>About Us</block:title>

<block:content>
   Something about us, <?=$someVariable?>. <a href="/some-url/">Some Address</a>.
</block:content>
```

> Attention, ViewManager does not know about changes in parent layouts, you should either recompile your views using the `views:compile` command or simply disable the cache in the development enviroment.

## Parent block content
In our previous example we demonstrated the ability to inject the desired values into our parent layout, however in many cases we might want to be able to rewrite specific parts of parent layout without losing the original content. We can do this by defining a few new blocks in our parent layout:

```html
<!DOCTYPE html>
<html>
<head>
    <title>
        <yield:title/>
    </title>
    <block:resources>
        <link rel="stylesheet" href="/resources/styles/website.css"/>
    </block:resources>
</head>
<body>
    <div class="header">
       <block:header>This is our header.</block:header>
    </div>
    <div class="wrapper">
        <yield:content/>
    </div>
    <div class="footer">
        <block:footer>This is our footer.</block:footer>
    </div>
</body>
</html>
```

If we will try to render our about us page now (do not forget to reset cache) we will notice no difference as no replacements come for newly created blocks. Let's try to overwrite value of layout footer:

```html
<extends:layouts.basic title="About Us"/>

<block:content>
     Something about us, <?=$someVariable?>. <a href="/some-url/">Some Address</a>.
</block:content>

<block:footer>
    This is about us specific footer.
</block:footer>
```

This methodic can work with some simple blocks, however in more complex cases it might look useless, let's try to imagine that we would to use additional CSS style for about us page. We can do that, by redefining block resources:

```html
<extends:layouts.basic title="About Us"/>

<block:resources>
   <link rel="stylesheet" href="/resources/styles/website.css"/>
   <link rel="stylesheet" href="/resources/styles/aboutus.css"/>
</block:resources>

<block:content>
    Something about us <?=$someVariable?>. <a href="/some-url/">Some Address</a>.
</block:content>

<block:footer>
    This is about us specific footer.
</block:footer>
```

Now we have a small problem, every time we would like to add new content into parent block we have to carry into content with us which can lead to some problems when new resources mounted to parent layout. We can solve this issue by injecting value of parent block into it's redefinition:

```html
<extends:layouts.basic title="About Us"/>

<block:resources>
   <block:resources/>
   <link rel="stylesheet" href="/resources/styles/aboutus.css"/>
</block:resources>

<block:content>
     Something about us, <?=$someVariable?>. <a href="/some-url/">Some Address</a>.
</block:content>

<block:footer>
    This is about us specific footer.
</block:footer>
```

We can inject parent block content into any place of re-definition, before, after or even in a middle of new content:

```html
<extends:layouts.basic title="About Us"/>

<block:resources>
    <link rel="stylesheet" href="/resources/styles/weird.css"/>
   <block:resources/>
   <link rel="stylesheet" href="/resources/styles/aboutus.css"/>
</block:resources>

<block:content>
    Something about us. <a href="/some-url/">Some Address</a>.
</block:content>

<block:footer>
    This is about us specific footer.
</block:footer>
```

Such technique provides us ability to manage set of layouts on global level with ability to add/replace needed assets on page template level.

> Templates compiles views before PHP code will be executed, you are not allow to control block structures with php code. Leave it for html/frontent developers.

## Even shorter block/yield definition
There is many scenarious when you would to inject child value inside html tag or even php code, let's try to say that we want to be able to define custom wrapper class for different views.

```html
<!DOCTYPE html>
<html>
<head>
    <title>
        <yield:title/>
    </title>
    <block:resources>
        <link rel="stylesheet" href="/resources/styles/website.css"/>
    </block:resources>
</head>
<body>
    <div class="header">
       <block:header>This is our header.</block:header>
    </div>
    <div class="wrapper <block:wrapper-class>page</block:wrapper-class>">
        <yield:content/>
    </div>
    <div class="footer">
        <block:footer>This is our footer.</block:footer>
    </div>
</body>
</html>
```

Unfortunatelly such definition is **not valid** as it violates html standards. In such case we can use fallback Templater construction which can be injected in almost any part of your template: `${name|default value}`, let's try to utilize it:

```html
<!DOCTYPE html>
<html>
<head>
    <title>
        <yield:title/>
    </title>
    <block:resources>
        <link rel="stylesheet" href="/resources/styles/website.css"/>
    </block:resources>
</head>
<body>
    <div class="header">
       <block:header>This is our header.</block:header>
    </div>
    <div class="wrapper ${wrapper-class|page}">
        <yield:content/>
    </div>
    <div class="footer">
        <block:footer>This is our footer.</block:footer>
    </div>
</body>
</html>
```

Now we have an ability to set custom wrapper class from our templates:

```html
<extends:layouts.basic title="About Us" wrapper-class="about-us"/>

<block:resources>
    <link rel="stylesheet" href="/resources/styles/weird.css"/>
   <block:resources/>
   <link rel="stylesheet" href="/resources/styles/aboutus.css"/>
</block:resources>

<block:content>
     Something about us, <?=$someVariable?>. <a href="/some-url/">Some Address</a>.
</block:content>

<block:footer>
    This is about us specific footer.
</block:footer>
```

As in case with normal block definitions we always use parent block value, let's try to add our class without removing parent one using `${name}` like syntax:

```html
<extends:layouts.basic title="About Us" wrapper-class="${wrapper-class} about-us"/>

<block:resources>
    <link rel="stylesheet" href="/resources/styles/weird.css"/>
   <block:resources/>
   <link rel="stylesheet" href="/resources/styles/aboutus.css"/>
</block:resources>

<block:content>
     Something about us, <?=$someVariable?>. <a href="/some-url/">Some Address</a>.
</block:content>

<block:footer>
    This is about us specific footer.
</block:footer>
```

## Multiple inheritance
Templater does not have any limitations (except memory :)) on how many times template can be extended, as result we are able to multiple website layouts, for example let's say that we want to create new layout with navigation, first of all, let's modify our basic layout to provide such ability:

```html
<!DOCTYPE html>
<html>
<head>
    <title>
        <yield:title/>
    </title>
    <block:resources>
        <link rel="stylesheet" href="/resources/styles/website.css"/>
    </block:resources>
</head>
<body>
<div class="header">
    <block:header>This is our header.</block:header>
</div>
<block:wrapper>
    <div class="wrapper ${wrapper-class|page}">
        <yield:content/>
    </div>
</block:wrapper>
<div class="footer">
    <block:footer>This is our footer.</block:footer>
</div>
</body>
</html>
```

We can now create new layout with navigation realted html, we can call it "layouts/navigation":

```html
<extends:layouts.basic/>

<block:wrapper>
    <div class="navigation">
        Some navigation.
    </div>
    <block:wrapper/>
</block:wrapper>
```

To apply new layout to our about-us page let's simply change it's extends tag:

```html
<extends:layouts.navigation title="About Us" wrapper-class="${wrapper-class} about-us"/>

<block:resources>
    <link rel="stylesheet" href="/resources/styles/weird.css"/>
    <block:resources/>
    <link rel="stylesheet" href="/resources/styles/aboutus.css"/>
</block:resources>

<block:content>
    Something about us, <?=$someVariable?>. <a href="/some-url/">Some Address</a>.
</block:content>

<block:footer>
    This is about us specific footer.
</block:footer>
```

As result we managed to change page layout using only one line of code.

> Templater fully supports view namespaces, as result we can define our extends tag in a form: `<extends:default:layouts.navigation/>`

## Switching layouts
Even if you are not allowed to control Templater structures and inheritance using PHP code - you achieve goal of switching between different templates by using view processors which are executed about Templater. 

First of all let's try to create new view dependency to declare if we using mobile or desktop version of website, to do that we have to modify view configuration:

```php
 'dependencies' => [
        'language' => [
            'i18n',
            'getLanguage'
        ],
        'basePath' => [
            'http',
            'basePath'
        ],
        'layout'   => [
            Application::class,
            'websiteLayout'
        ]
    ],
```

Inside our application we have to create method `websiteLayout`:

```php
/**
 * Website layout to be used.
 * 
 * @return string
 */
public function websiteLayout()
{
    return 'desktop';
}
```

Now, we can use alternative syntax to define extends tag which gives us ability to provide parent layout name in attribute form (use view:namespace to change parent layout namespace same way):

```html
<extends view:parent="layouts/desktop"/>
```

If already read about ViewManager, it's cache dependencies and processors, you might remember that you can inject value of cache dependency into html using `@{name}` syntax, such injection will happen before Tempalter will start processing our views as result we can rewrite our extends tag to follow value declared in such dependency:

```html
<extends view:parent="layouts/@{layout}"/>
```

Rendered template will look exactly as before, however now we can try to create different layout with new markup and resources, let's put into "layouts/mobile":

```html
<!DOCTYPE html>
<html>
<head>
    <title>
        <yield:title/>
    </title>
    <block:resources>
        <link rel="stylesheet" href="/resources/styles/mobile.css"/>
    </block:resources>
</head>
<body>
<div class="mobile">
    <yield:content/>
</div>
</body>
</html>
```

Now, the only thing we have to do to change website/page layout is provide different value from our `Application->websiteLayout` method.

```php
/**
 * Website layout to be used.
 *
 * @return string
 */
public function websiteLayout()
{
    return 'mobile';
}
```

> Interesting side effect of using cache dependencies to switch layouts, that such switch will happen on compilation stage, as result you will get different template cache versions for different values of `websiteLayout` method. We can always go into "application/runtime/cache/views" folder and locate two diffent files related to our page.

```html
<!DOCTYPE html>
<html>
<head>
    <title>
        About Us
    </title>
    <link rel="stylesheet" href="/resources/styles/weird.css"/>
        <link rel="stylesheet" href="/resources/styles/website.css"/>
    <link rel="stylesheet" href="/resources/styles/aboutus.css"/>
</head>
<body>
<div class="header">
    This is our header.
</div>
    <div class="wrapper page about-us">
    Something about us, <?=$someVariable?>. <a href="/some-url/">Some Address</a>.
    </div>
<div class="footer">
    This is about us specific footer.
</div>
</body>
</html>
```

```html
<!DOCTYPE html>
<html>
<head>
    <title>
        About Us
    </title>
    <link rel="stylesheet" href="/resources/styles/weird.css"/>
        <link rel="stylesheet" href="/resources/styles/mobile.css"/>
    <link rel="stylesheet" href="/resources/styles/aboutus.css"/>
</head>
<body>
<div class="mobile">
    Something about us, <?=$someVariable?>. <a href="/some-url/">Some Address</a>.
</div>
</body>
</html>
```

> You can move `websiteLayout` method to any of desired class including services and controllers.
