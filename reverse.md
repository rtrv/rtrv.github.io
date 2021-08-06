# How to reverse an array

There's a standard method `Array#reverse` in Ruby. Implementing of standard methods is often used for education in software development: it's convenient to check when you've got the reference solution. And there are multiple ways to implement it. I'm gonna try to find as many solutions as possible and choose my own favorite.

I will collect options from the following sources:

1. I found such a [kata](https://www.codewars.com/kata/53da6d8d112bd1a0dc00008b) on Codewars. So I'll track my solution until it's ready
2. Other people solutions from Codewars
2. StackOvervlow
3. Standard library implementation
4. Community help
  
Then let's benchmark it.

## Solving the kata

The kata has the lowest complexity rank: 8 kyu. There's no limit of using standard `Array#reverse`, so it's a valid solution:
 
```ruby
def reverse_list(list)
  list.reverse
end
```

Then you can refactor your solution before you submit it and get access to other solutions. Let's assume that there's no `Array#reverse` and move on.

The first solution is based on the fact that we can iterate over the list and use accumulative array:
 
```ruby
def reverse_list(list)
  reversed_list = []
	
  for n in list
    reversed_list.unshift(n)
  end
  
  reversed_list
end
```
 
In fact it's a reduction process, so let's rewrite it with `Enumerable#reduce` with accumulator:
 
```ruby
def reverse_list(list)
  list.reduce([]) { |acc, n| acc.unshift(n) }
end
```

At this point I've got the following ways to make it better:

1. Get rid of mutating accumulative array and unshifting
2. Use some better alternative for `Array#unshift` if there's one

### Getting rid of mutating accumulator

In this case I have to initialize reversed array once with all the members of it. Let's give it a shot:

```ruby
def reverse_list(list)
  Array.new(list.length) { |i| list[list.length - i] }
end
```

Here I used an ability to pass a block into array constructor.

### `Array#unshift` alternatives

There's a straight-forward one:

```ruby
def reverse_list(list)
  list.reduce([]) { |acc, n| [n] + acc }
end
```

`Array#unshift` is written in C so I'm quite sure that there's no way to find some more effitient way to mutate an array over iterator adding elements into array head. Here's it's source code:

```c
static VALUE
rb_ary_unshift_m(int argc, VALUE *argv, VALUE ary)
{
    long len = RARRAY_LEN(ary);
    VALUE target_ary;

    if (argc == 0) {
        rb_ary_modify_check(ary);
        return ary;
    }

    target_ary = ary_ensure_room_for_unshift(ary, argc);
    ary_memcpy0(ary, 0, argc, argv, target_ary);
    ARY_SET_LEN(ary, len + argc);
    return ary;
}
```

## Codewars best solutions

All the best solutions there are made using standard `Array#reverse`. However there are a few alternatives marked as "Clever". The first one uses map with index and looks quite good:

```ruby
def reverse_list(list)
  
  list.map.with_index{|num, idx| list[(list.length - 1) - idx] }
  
end
```

Another one uses accumulator and `Array#pop` method:

```ruby
def reverse_list(list)
  # list.reverse
  result = []
  while list.length != 0
    result << list.pop
  end
  result
end
```

## StackOverflow

Nothing special in fact. The majority of answers are about using `Array#reverse` and some bad examples of iteratation over an array with mutating accumulator or incremental index to access array elements.

## Standard library

It's obviously written in C. It reservates memory for the same length array and duplicates an array with reversed pointers:

```c
static VALUE
rb_ary_reverse_m(VALUE ary)
{
    long len = RARRAY_LEN(ary);
    VALUE dup = rb_ary_new2(len);

    if (len > 0) {
        const VALUE *p1 = RARRAY_CONST_PTR_TRANSIENT(ary);
        VALUE *p2 = (VALUE *)RARRAY_CONST_PTR_TRANSIENT(dup) + len - 1;
        do *p2-- = *p1++; while (--len > 0);
    }
    ARY_SET_LEN(dup, RARRAY_LEN(ary));
    return dup;
}
```

## Lenth times based solution

```ruby
def reverse_list(list)
  list.length.times { |n|   }
end
```

## Benchmark

Let's create simple benchmarking setup:

```ruby
class ReverseBenchmark
  TEST_LIST_LENGTH = 100000

  def call(length = TEST_LIST_LENGTH)
    methods = self.methods.grep /reverse_/

    methods.map { |m| benchmark(m, length) }
  end

  # Reverse implementations

  def reverse_standard(list)
    list.reverse
  end

  def reverse_list_for(list)
    reversed_list = []

    for n in list
      reversed_list.unshift(n)
    end

    reversed_list
  end

  def reverse_reduce_unshift(list)
    list.reduce([]) { |acc, n| acc.unshift(n) }
  end

  def reverse_new_array(list)
    Array.new(list.length) { |i| list[list.length - i] }
  end

  def reverse_reduce_sum(list)
    list.reduce([]) { |acc, n| [n] + acc }
  end

  def reverse_map_with_index(list)
    list.map.with_index{|num, idx| list[(list.length - 1) - idx] }
  end

  def reverse_length_times(list)
    list.length.times { |n| list[list.length - n] }
  end

  def reverse_half_length_times(list)
    length = list.length
    (0..((length - 1) / 2)).each do |i|
      n = list[i]
      list[i] = list[length - i - 1]
      list[length - i - 1] = n
    end

    list
  end

  private

  def benchmark(method, length)
    test_list = generate_list(length)

    before = Time.now
    self.send(method, test_list)
    after = Time.now

    puts "#{(after - before).round(5)} seconds has been spent to reverse an array with #{length} elements by #{method}"
  end

  def generate_list(length)
    Array.new(length) { rand() }
  end
end

ReverseBenchmark.new.call
```

There's an example of the result:

```bash
0.0004 seconds has been spent to reverse an array with 100000 elements by reverse_standard
0.00884 seconds has been spent to reverse an array with 100000 elements by reverse_list_for
0.01205 seconds has been spent to reverse an array with 100000 elements by reverse_reduce_unshift
0.0071 seconds has been spent to reverse an array with 100000 elements by reverse_new_array
5.98098 seconds has been spent to reverse an array with 100000 elements by reverse_reduce_sum
0.01047 seconds has been spent to reverse an array with 100000 elements by reverse_map_with_index
0.00616 seconds has been spent to reverse an array with 100000 elements by reverse_length_times
0.00607 seconds has been spent to reverse an array with 100000 elements by reverse_half_length_times
```

## Conclusions

1. Standard `Array#reverse` works at least ~10-20 times faster than any Ruby-based implementation
2. Iterate over an array with unshifting elements works as good as map with index. `reverse_list_for` is a bit faster.
3. `Array.new` was a good idea to make it a bit faster. It was based on the idea to get rid of array mutations
4. Replacing of `unshift` to `[n] + acc` dramatically reduced performance
5. If there's a proper method in Ruby standard library, you better use it
6. My personal favorite was the one with `unshift`
7. Iterate over array length or half of length are the best Ruby `reverse` implementations in terms of performance. Both ideas were provided at https://t.me/saintprug
