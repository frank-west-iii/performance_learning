class: center, middle

# Ruby Performance

## Frank West
---

# What makes Ruby slow?

It is common knowledge that Ruby is slow but....

* why is it slow?
* what is making it faster every version?

---

# Base Code

```ruby
require 'benchmark'

num_rows = 100000
num_columns = 10
data = Array.new(num_rows) { Array.new(num_columns) { 'x'*1000 } }

puts "Memory Before: %d MB" % (`ps -o rss= -p #{Process.pid}`.to_i/1024)

time = Benchmark.realtime do
  csv = data.map { |row| row.join(',') }.join("\n")
end

puts "Memory After: %d MB" % (`ps -o rss= -p #{Process.pid}`.to_i/1024)

puts "Time to run: #{time.round(2)} seconds"

```

---

# 1.9.3 (-p551)

```bash
Memory Before: 629 MB
Memory After: 1446 MB
Time to run: 10.35 seconds
```

---

# 2.0.0 (-p645)

```bash
Memory Before: 1067 MB
Memory After: 1565 MB
Time to run: 11.01 seconds
```

---

# 2.1.6

```bash
Memory Before: 1058 MB
Memory After: 1341 MB
Time to run: 4.85 seconds
```

---

# 2.2.2

```bash
Memory Before: 1060 MB
Memory After: 1534 MB
Time to run: 4.74 seconds
```
---

# So there we go

--

# But where do these gains come from?

--

# Let's show you

---

# Modified Code

```ruby
require 'benchmark'

num_rows = 100000
num_columns = 10
data = Array.new(num_rows) { Array.new(num_columns) { 'x'*1000 } }

puts "Memory Before: %d MB" % (`ps -o rss= -p #{Process.pid}`.to_i/1024)

GC.disable

time = Benchmark.realtime do
  csv = data.map { |row| row.join(',') }.join("\n")
end

puts "Memory After: %d MB" % (`ps -o rss= -p #{Process.pid}`.to_i/1024)

puts "Time to run: #{time.round(2)} seconds"

```
---

# 1.9.3 (-p551)

```bash
Memory Before: 1032 MB
Memory After: 1608 MB
Time to run: 3.25 seconds
```

---

# 2.0.0 (-p645)

```bash
Memory Before: 1066 MB
Memory After: 1483 MB
Time to run: 3.93 seconds
```

---

# 2.1.6

```bash
Memory Before: 1060 MB
Memory After: 1562 MB
Time to run: 3.36 seconds
```

---

# 2.2.2

```bash
Memory Before: 1061 MB
Memory After: 1573 MB
Time to run: 3.48 seconds
```
---

# We can't disable GC

--

## :( so what do we do?

--

## We need to optimize the code in other ways

---

```ruby
require 'benchmark'

num_rows = 100000
num_columns = 10
data = Array.new(num_rows) { Array.new(num_columns) { 'x'*1000 } }

puts "Memory Before: %d MB" % (`ps -o rss= -p #{Process.pid}`.to_i/1024)

time = Benchmark.realtime do  |x|
  csv = ''
  num_rows.times do |i|
    num_columns.times do |j|
      csv << data[i][j]
      csv << ',' unless j == num_columns - 1
    end
    csv << "\n" unless i == num_rows - 1
  end
end

puts "Memory After: %d MB" % (`ps -o rss= -p #{Process.pid}`.to_i/1024)

puts "Time to run: #{time.round(2)} seconds"
```

---

# 1.9.3 (-p551)

```bash
Memory Before: 639 MB
Memory After: 1055 MB
Time to run: 2.75 seconds
```

---

# 2.0.0 (-p645)

```bash
Memory Before: 1066 MB
Memory After: 843 MB
Time to run: 2.47 seconds
```

---

# 2.1.6

```bash
Memory Before: 1059 MB
Memory After: 1205 MB
Time to run: 2.04 seconds
```

---

# 2.2.2

```bash
Memory Before: 1059 MB
Memory After: 1265 MB
Time to run: 1.91 seconds
```
---

# Strings

str = 'X' * 1024 * 1024 * 10 # 10 MB string

str = str.downcase # Allocates another 10 MB of memory

str.downcase! # Does not allocate any more memory

---

# Arrays

data = Array.new(100) { 'x' * 1024 * 1024 }

data.map { |str| str.upcase }  # Allocates another 16 MB of memory

data.map! { |str| str.upcase! } # Does not allocate any more memory

---

# Remove objects from collection during iteration

Removing objects from your collection as you iterate will help improve your overall memory footprint and execution time.

Removing objects is not possible using iteration operators such as .each, .each_index, etc.. The entire collection with remain in memory until the iteration is completed and the collection is deallocated manually or falls out of scope.

--

```ruby
list = Array.new(100000) { Thing.new }

list.each do |obj|
 puts obj
end

# Number of objects available to GC 0 here
list = nil
# Number of objects available to GC is now 100000
```
---
One possible solution is to remove the items as you iterate using while loops or for loops.

```ruby
list = Array.new(100000) { Thing.new }

while list.count > 0
  obj = list.shift
end

# Each iteration of the loop makes an additional object available to GC

```
---

# Be cautious about iterators that create additional objects

Some iterators create additional T_NODE objects within the implementation of the iterator. This can become an issue when you use these iterators nested within a large collection iteration.

In the example below, the .all? operator creates 3 additional T_NODE objects when it is used. This results in the total number of additional objects being equal to size_of_array * 3.

```ruby
GC.disable

before = ObjectSpace.count_objects
list = Array.new(10000) { Array.new(1) }

list.each do |i|
  i.all? {}
end

after = ObjectSpace.count_objects
puts '# of nodes: %d' % (after[:T_NODE] - before[:T_NODE]) #=> 30000
```

---

# Be cautious of inefficient code within the iteration

Using certain code within a large collection can significantly slow down your execution time.

Date Parsing as an example:

```ruby
require 'date'
require 'benchmark/ips'

date = '2014-05-23'
Benchmark.ips do |x|
  x.report('Date.parse') { Date.parse(date) }
  x.report('Date.strptime') { Date.strptime(date, '%Y-%m-%d') }
  x.report('Date.civil') {
    Date.civil(date[0,4].to_i, date[5,2].to_i, date[8,2].to_i) }

  x.compare!
end
```
---
Results:
```bash
Calculating -------------------------------------
          Date.parse     9.835k i/100ms
       Date.strptime    33.779k i/100ms
          Date.civil    49.702k i/100ms
-------------------------------------------------
          Date.parse    114.594k (± 4.1%) i/s -    580.265k
       Date.strptime    499.884k (± 2.5%) i/s -      2.500M
          Date.civil    885.387k (± 2.2%) i/s -      4.473M
Comparison:
          Date.civil:   885386.7 i/s
       Date.strptime:   499883.7 i/s - 1.77x slower
          Date.parse:   114594.1 i/s - 7.73x slower
```
---
Thank you