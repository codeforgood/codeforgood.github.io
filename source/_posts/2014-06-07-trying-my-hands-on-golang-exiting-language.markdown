---
layout: post
title: "Hands on Golang"
date: 2014-06-07 16:44:55 -0400
comments: true
categories: [Go, parallel computing, languages]
author: Sadish Ravi
---
I am starting to learn Go and I feel the best way to learn is by sharing. Lets go through a simple example to demonstrate the power of Go

##### Problem - Sum of numbers from 1 to N
To explore the power of Go we will not use [Summation series](http://en.wikipedia.org/wiki/1_%2B_2_%2B_3_%2B_4_%2B_%E2%8B%AF "Summation Series")

Typical way to sum 1 to N numbers in Python
{% codeblock Sum of numbers from 1 to 99 in python - sum.go %}
def sum(a, b):
    sum = 0
    for i in range(a, b):
        sum += i
    return sum

print sum(1, 100)
{% endcodeblock %}

The above same problem when translated to Go looks like this
{% codeblock Sum of numbers from 1 to 99 in Go (serial version) - sum_serial.go %}
package main

import "fmt"

func sum(a int, b int) (int) {
    sum := 0
    for i := a; i < b; i++ {
        sum += i
    }
    return sum
}

func main() {
    result := sum(1, 100)
    fmt.Println(result)
}
{% endcodeblock %}
Both the above solutions has linear complexity - O(n), runs in a single thread What if we want to sum from 1 to 1 million ???

Go provides a way to do this computation much faster and easier using [goroutines](https://gobyexample.com/goroutines) and [channels](https://gobyexample.com/channels)

    - The above sum function can be changed to return a channel with a result once available
    - The main method can fork multiple sum methods as goroutines asking to sum different ranges
    - The main method will grab the results from the channels and will return the sum

Theoretically, If we divide the array into 2 halves and sum using dedicated goroutines, we can get a speed up of twice, assuming multiple cores are available to run the routines in parallel

The below go snippet shows the usage of goroutines and channels to achieve parallel computation
{% codeblock Sum of numbers from 1 to 99 in Go (parallel version) - sum_parallel.go %}
package main

import "fmt"

func sum(start int, end int) <-chan int{
    out := make(chan int)
    //goroutine
    go func(){
        sum := 0
        for i := start; i < end; i++ {
            sum += i
        }
        out <- sum
        close(out)
    }()
    return out
}

func main() {
    c1 := sum(1, 50) // call to sum returns immediately with a channel
    c2 := sum(50, 100)
    sum_first_half, sum_second_half := <-c1, <-c2
    fmt.Println(sum_first_half + sum_second_half)
}
{% endcodeblock %}

***

##### References

Really interesting article about why [Go is faster than other languages](http://dave.cheney.net/2014/06/07/five-things-that-make-go-fast)
