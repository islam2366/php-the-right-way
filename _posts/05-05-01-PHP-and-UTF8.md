---
title:   UTF-8 নিয়ে কাজ করা
isChild: true
anchor:  php_and_utf8
---

## UTF-8 নিয়ে কাজ করা {#php_and_utf8_title}

_এই অংশটি মূলত [Alex Cabal](https://alexcabal.com/) এর লিখা থেকে হুবহু অনুবাদ করা হয়েছে যা [PHP Best Practices]
(https://phpbestpractices.org/#utf-8) এ পাওয়া যাবে এবং এটি UTF-8 এর জন্য আমাদের নিজস্ব উপদেশগুলোর মূল ভিত্তি হিসেবেও কাজ করছে।_

### There's no one-liner. Be careful, detailed, and consistent.

এখন PHP লো লেভেল এর ইউনিকোড সমর্থন করে না। UTF-8 স্ট্রিংগুলি ঠিক আছে কিনা তা নিশ্চিত করার কিছু উপায় রয়েছে,
তবে তা খুব সহজ না, এবং এর জন্য ওয়েব অ্যাপ এর প্রায় সকল স্তর সম্বন্ধে ভাল ধারণা থাকা প্রয়োজন, HTML থেকে শুরু করে SQL এমনকি PHP সম্বন্ধেও।
আমরা এখানে খুব সংক্ষেপে এগুলো আলোচনা করব।

### PHP লেভেলে UTF-8 

বেসিক string অপারেশন, যেমন দুইটি string এর concatenation এবং string কে ভেরিয়েবল এ অ্যাসাইন করার জন্য UTF-8 এর আলাদা কোন কিছুর
প্রয়োজন হয় না। তবে, বেশিরভাগ string ফাংশন, যেমন `strpos()` এবং `strlen()` এর ক্ষেত্রে বিশেষ নিয়ম মেনে চলতে হয়। এসব ফাংশনে অনেক
সময় `mb_*` অংশটি যুক্ত থাকেঃ উদাহরণস্বরূপ, `mb_strpos()` এবং `mb_strlen()`। এসব `mb_*` string গুলো [Multibyte String Extension]
এর মাধ্যমে পাওয়া যায় এবং এগুলো Unicode string এর সাথে ব্যবহার উপযোগী করেই ডিজাইন করা হয়েছে।

যখনই আপনি কোন Unicode string নিয়ে কাজ করবেন, আপনাকে অবশ্যই `mb_*` ফাংশন ব্যবহার করতে হবে। উদাহরণস্বরূপ, আপনি যদি কোন UTF-8 string 
এ `substr()` ব্যবহার করেন, তবে আউটপুটে কিছু আবদ্ধ অর্ধ-অক্ষর থাকার সম্ভাবনা অনেক বেশি। এক্ষেত্রে সঠিক আউটপুটের জন্য উক্ত ফাংশনের বদলে
`mb_substr()` ব্যবহার করাই শ্রেয়।

The hard part is remembering to use the `mb_*` functions at all times. If you forget even just once, your Unicode
string has a chance of being garbled during further processing.

Not all string functions have an `mb_*` counterpart. If there isn't one for what you want to do, then you might be out
of luck.

You should use the `mb_internal_encoding()` function at the top of every PHP script you write (or at the top of your
global include script), and the `mb_http_output()` function right after it if your script is outputting to a browser.
Explicitly defining the encoding of your strings in every script will save you a lot of headaches down the road.

Additionally, many PHP functions that operate on strings have an optional parameter letting you specify the character
encoding. You should always explicitly indicate UTF-8 when given the option. For example, `htmlentities()` has an
option for character encoding, and you should always specify UTF-8 if dealing with such strings. Note that as of PHP 5.4.0, UTF-8 is the default encoding for `htmlentities()` and `htmlspecialchars()`.

Finally, If you are building a distributed application and cannot be certain that the `mbstring` extension will be
enabled, then consider using the [patchwork/utf8] Composer package. This will use `mbstring` if it is available, and
fall back to non UTF-8 functions if not.

[Multibyte String Extension]: http://php.net/book.mbstring
[patchwork/utf8]: https://packagist.org/packages/patchwork/utf8

### ডেটাবেস লেভেলে UTF-8 

If your PHP script accesses MySQL, there's a chance your strings could be stored as non-UTF-8 strings in the database
even if you follow all of the precautions above.

To make sure your strings go from PHP to MySQL as UTF-8, make sure your database and tables are all set to the
`utf8mb4` character set and collation, and that you use the `utf8mb4` character set in the PDO connection string. See
example code below. This is _critically important_.

Note that you must use the `utf8mb4` character set for complete UTF-8 support, not the `utf8` character set! See
Further Reading for why.

### ব্রাউজার লেভেলে UTF-8 

PHP স্ক্রিপ্ট এর মাধ্যমে ব্রাউজারে UTF-8 string এর সঠিক আউটপুট নিশ্চিতকরণের জন্য `mb_http_output()` ফাংশনটি ব্যবহার করুন।

এরপর HTTP response এর মাধ্যমে ব্রাউজারকে জানিয়ে দিতে হবে যে এই পেইজটিতে UTF-8 ব্যবহৃত হয়েছে। এটি করার জন্য যে পদ্ধতিটি বহুলভাবে ব্যবহৃত হয়ে
আসছে তা হল আপনার পেইজ এর `<head>` ট্যাগে [charset `<meta>` tag](http://htmlpurifier.org/docs/enduser-utf8.html) সংযোজন করা।
এই পদ্ধতিটি একেবারেই সঠিক, তবে `Content-Type` header এ charset সেটিং করার মাধ্যমে এটি
[অনেক তাড়াতাড়ি](https://developers.google.com/speed/docs/best-practices/rendering#SpecifyCharsetEarly) করা সম্ভব।

{% highlight php %}
<?php
// Tell PHP that we're using UTF-8 strings until the end of the script
mb_internal_encoding('UTF-8');
 
// Tell PHP that we'll be outputting UTF-8 to the browser
mb_http_output('UTF-8');
 
// Our UTF-8 test string
$string = 'Êl síla erin lû e-govaned vîn.';
 
// Transform the string in some way with a multibyte function
// Note how we cut the string at a non-Ascii character for demonstration purposes
$string = mb_substr($string, 0, 15);
 
// Connect to a database to store the transformed string
// See the PDO example in this document for more information
// Note the `charset=utf8mb4` in the Data Source Name (DSN)
$link = new PDO(
    'mysql:host=your-hostname;dbname=your-db;charset=utf8mb4',
    'your-username',
    'your-password',
    array(
        PDO::ATTR_ERRMODE => PDO::ERRMODE_EXCEPTION,
        PDO::ATTR_PERSISTENT => false
    )
);
 
// Store our transformed string as UTF-8 in our database
// Your DB and tables are in the utf8mb4 character set and collation, right?
$handle = $link->prepare('insert into ElvishSentences (Id, Body) values (?, ?)');
$handle->bindValue(1, 1, PDO::PARAM_INT);
$handle->bindValue(2, $string);
$handle->execute();
 
// Retrieve the string we just stored to prove it was stored correctly
$handle = $link->prepare('select * from ElvishSentences where Id = ?');
$handle->bindValue(1, 1, PDO::PARAM_INT);
$handle->execute();
 
// Store the result into an object that we'll output later in our HTML
$result = $handle->fetchAll(\PDO::FETCH_OBJ);

header('Content-Type: text/html; charset=UTF-8');
?><!doctype html>
<html>
    <head>
        <meta charset="UTF-8">
        <title>UTF-8 test page</title>
    </head>
    <body>
        <?php
        foreach($result as $row){
            print($row->Body);  // This should correctly output our transformed UTF-8 string to the browser
        }
        ?>
    </body>
</html>
{% endhighlight %}

### আরও পড়ুন

* [PHP Manual: String Operations](http://php.net/language.operators.string)
* [PHP Manual: String Functions](http://php.net/ref.strings)
    * [`strpos()`](http://php.net/function.strpos)
    * [`strlen()`](http://php.net/function.strlen)
    * [`substr()`](http://php.net/function.substr)
* [PHP Manual: Multibyte String Functions](http://php.net/ref.mbstring)
    * [`mb_strpos()`](http://php.net/function.mb-strpos)
    * [`mb_strlen()`](http://php.net/function.mb-strlen)
    * [`mb_substr()`](http://php.net/function.mb-substr)
    * [`mb_internal_encoding()`](http://php.net/function.mb-internal-encoding)
    * [`mb_http_output()`](http://php.net/function.mb-http-output)
    * [`htmlentities()`](http://php.net/function.htmlentities)
    * [`htmlspecialchars()`](http://php.net/function.htmlspecialchars)
* [PHP UTF-8 Cheatsheet](http://blog.loftdigital.com/blog/php-utf-8-cheatsheet)
* [Handling UTF-8 with PHP](http://www.phpwact.org/php/i18n/utf-8)
* [Stack Overflow: What factors make PHP Unicode-incompatible?](http://stackoverflow.com/questions/571694/what-factors-make-php-unicode-incompatible)
* [Stack Overflow: Best practices in PHP and MySQL with international strings](http://stackoverflow.com/questions/140728/best-practices-in-php-and-mysql-with-international-strings)
* [How to support full Unicode in MySQL databases](http://mathiasbynens.be/notes/mysql-utf8mb4)
* [Bringing Unicode to PHP with Portable UTF-8](http://www.sitepoint.com/bringing-unicode-to-php-with-portable-utf8/)
* [Stack Overflow: DOMDocument loadHTML does not encode UTF-8 correctly](http://stackoverflow.com/questions/8218230/php-domdocument-loadhtml-not-encoding-utf-8-correctly)
