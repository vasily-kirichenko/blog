---
title: "Use the right tool for the job"
date: 2021-10-27T17:48:30+03:00
draft: true
---
The task: store N recent values with fast access to the oldest one. This is the definition of circular buffer, but we don't 
want to implement it ourselves, so we try to use a slice:
```go
const N = 1000

func BenchmarkSlice(b *testing.B) {
	b.ReportAllocs()
	var s []int
	for i := 0; i < N; i++ {
		s = append(s, i)
	}
	b.ResetTimer()
	for n := 0; n < b.N; n++ {
		for i := 0; i < 10*N; i++ {
			s = s[1:]
			s = append(s, i)
		}
	}
}
```
which performs not great, because the underlying array is constantly reallocated (O(log(n))?)
```
BenchmarkSlice  33313  31647 ns/op  156187 B/op  9 allocs/op
```
Well, it was expected. Ah, we have a nice linked list implementation right in the standard library! 
Let's use it in hope to reduce allocations:
```go
func BenchmarkList(b *testing.B) {
	b.ReportAllocs()
	l := list.New()
	for i := 0; i < N; i++ {
		l.PushBack(i)
	}
	b.ResetTimer()
	for n := 0; n < b.N; n++ {
		for i := 0; i < 10*N; i++ {
			l.Remove(l.Front())
			l.PushBack(i)
		}
	}
}
```
The result is rather surprising:
```
BenchmarkList  2226	 601889 ns/op  557953 B/op  19744 allocs/op
```
Uh :( Interesting. What about the right structure?
```go
func BenchmarkCircularBuffer(b *testing.B) {
	b.ReportAllocs()
	var buf []int
	for i := 0; i < N; i++ {
		buf = append(buf, i)
	}
	b.ResetTimer()
	for n := 0; n < b.N; n++ {
		var p int
		for i := 0; i < 10*N; i++ {
			buf[p] = i
			p++
			if p >= N {
				p = 0
			}
		}
	}
}
```
```
BenchmarkCircularBuffer  146463  8591 ns/op  0 B/op  0 allocs/op
```
All results in one table:
```
BenchmarkCircularBuffer
BenchmarkCircularBuffer-4   	  146463	      8591 ns/op	       0 B/op	       0 allocs/op
BenchmarkSlice
BenchmarkSlice-4            	   33313	     31647 ns/op	  156187 B/op	       9 allocs/op
BenchmarkList
BenchmarkList-4             	    2226	    601889 ns/op	  557953 B/op	   19744 allocs/op
```
So, linked list is 19 times slower that slice, and 70 times slower than circular buffer. 
It also allocates like crazy.

Conclusion: use the right tool for the job :)