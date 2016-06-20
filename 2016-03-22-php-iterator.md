---
layout: post
title: PHP iterator - inconsistence and how to trick it
categories:
- blog
- PHP
- technical
---

## PHP iterator: inconsistence and how to trick it

#### Motivation

>> "[The data structures in the SPL are flawed in many ways.](https://wiki.php.net/rfc/spl-improvements/data-structures)"

I've currently run across some difficulties with [PHP iterators](http://php.net/manual/en/class.iterator.php). It came to my attention after I fixed a code-passage in which I had to iterate over an array in reverse.
Given below is a brief example

```php
$blog = array(
    '2015' => array(
        'post1',
        'post2'
    ),
    '2016' => array(
        'post1',
        'post2'
    )  
);

// Loop through the array
foreach($blog as $year => $posts) {
    foreach($posts as $post) {
        echo $post . " was written in year " . $year;
    }
}

```

#### The problem
When you want to iterate reversively in PHP you will come across ```array_reverse``` at first. What you don't know is, it works multi-dimensional, meaning you will get an interesting result

```php
foreach(array_reverse($blog) as $year => $posts) {
    foreach($posts as $post) {
        echo $post . " was written in year " . $year;
    }
}

/*
post2 was written in year 2016
post1 was written in year 2016
post2 was written in year 2015
post1 was written in year 2015
*/
```
Since there is no explicit ```$options``` this is to be evaluated as [undefined behavoir](https://eev.ee/blog/2012/04/09/php-a-fractal-of-bad-design/).
Next on the list we can use traditional ```current(), next(), prev()``` functions to address mentioned problem. A solution may look similliar to this:

```php
end($blog);

for ($i = count($blog); $i > 0; $i--) {
    $key = key($blog);
    $value = current($blog);
    prev($blog);
} /* This is possible with a while loop, but leads to an issue
while (key($blog) > 0) {
    prev($blog);
}
... but this will only work with numeric indexes, such as in SPLFixedArray
*/
```

### Addressing the origin of the issue
Digging further into [```Iterator```](http://php.net/manual/en/class.iterator.php) leads us to the origin of mentioned problem.
I figured to miss the following things on the list, beeing well-known in other programming languages

 - there is no ```Iterator->null``` meaning it is not possible to retrieve the start-index of a list in PHP (or compare, since we are talking about dynamically typed languages)
 - neither is there ```Iterator->end``` which makes it impossible to get the end-index, offering no more possibilities than [```Countable```](http://php.net/manual/en/class.countable.php) to us
 - consequently you can say it is not possible to use ```Iterator``` as to what it is supposed: store a container index and make it comparable

With PHP lacking these (and more) essential Iterator concepts it is impossible to iterate with complex data storages (everything beside the mentioned [```SPLFixedArray```](http://php.net/manual/en/class.splfixedarray.php)). 

### How to trick iterators with ArrayAccess
However, PHP shows off as a swiss-knife-army once again. By abusing the logic of [```ArrayAccess```](
http://php.net/manual/en/class.arrayaccess.php) one can iterate through any container that implements the SPL interface.

```php
$blog = new ArrayObject(
    '2015' => array(
        'post1',
        'post2'
    ),
    '2016' => array(
        'post1',
        'post2'
    ) 
);

$index = $blog->count();
while ($blog->offsetExists($index)) {
    $posts = $blog->offsetGet($index);
    foreach($posts as $post) {
        echo $post . " was written in year " . $blog->key($posts);
    }
    
    $i--; // do this until no more index exists
} // this could be done with non-integer indexes as well!
```
Further synopsis can be found [at this article](http://www.phpro.org/tutorials/Introduction-to-SPL-ArrayAccess.html).
