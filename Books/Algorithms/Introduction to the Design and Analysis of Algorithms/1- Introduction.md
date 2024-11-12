There are both practical and theoretical reasons to study algorithms.
**From a practical standpoint**, I have to know a standard set of important algorithms from different areas of computing; in addition, I should be able to design new algorithms and analyze their efficiency. From the theoretical standpoint, the study if algorithms, sometimes called algorithmics, has come to be recognized as the cornerstone of computer science.

>_Algorithmics is more than a branch of computer science. It is the core of computer science, and, in all fairness, can be said to be relevant to most of_ science, business, and technology. [Har92, p. 6] - Algorithmics: the Spirit of Computing

Develop analytical skills is another reason to studying algorithms.

### 1.1 What is an algorithm?
It is a sequence of unambiguous  instructions for solving a problem, i.e., for obtaining a required output for any legitimate input in a finite amount of time.
![[algorithm.png]]

Recall that the greatest common divisor of two nonnegative, not-both-zero integers `m` and `n`, denoted `gcd(m, n)` is defined as the largest integer that divides both `m` and `n` evenly, i.e., with a remainder of zero.
Euclid of Alexandria outlined an algorithm for solving this problem in one of the volumes of his *Elements* most famous for its systematic exposition of geometry. In modern terms, *Euclid's algorithm* is based on applying repeatedly the equality.

`gcd(m, n) = gcd(n, m mod n)`,

where `m` mod `n` is the remainder of the division of `m` by `n`, until `m` mod `n` is equal to 0. Since `gcd(m, 0) = m` (why?), the last value of `m` is also the greatest common divisor of the initial `m` and `n`. For example, `gcd(60, 24)` can be computed as follows:
`gcd(60, 24) = gcd(24, 12) = gcd(12, 0) = 12`

**Euclid's algorithm** for computing `gcd(m, n)`:
1. if `n = 0`, return the value of `m` as the answer and stop; otherwise, proceed to Step 2.
2. divide `m` by `n` and assign the value of the remainder to `r`
3. assign the value of `n` to `m` and the value of `r` to `n`. Go to step 1.
In Go, it can be represented by this code:
```go
package main

import "fmt"

func gcd(m int, n int) int {
	for n != 0 {
		r := m % n
		m = n
		n = r
	}

	return m
}

func main() {
	r := gcd(60, 24)
	fmt.Println(r)
}
```

Of course, there are several algorithms for computing the greatest common divisor. I'll take a look at the other two methods for this problem.
The first is based on the definition of the greatest common divisor of `m` and `n` as the largest integer that divides both numbers evenly. Such a common divisor cannot be greater than the smaller of these numbers, which I will denote by `t = min{m, n}`. So I can start by checking whether `t` divides both `m` and `n`: if it does, `t` is the answer; if it does not, I simply decrease `t` by 1 and try again. For example, for numbers 60 and 24, the algorithm will try first 24, then 23, and so on, until it reaches 12, where it stops. 

**Consecutive integer checking algorithm** for computing `gcd(m, n)`
1. assign the value of `min{m, n}` to `t`
2. divide `m` by `t`. If the remainder of this division is `0`, go to step 3; otherwise, go to step 4.
3. divide `n` by `t`. If the remainder of this division is 0, return the value of `t` as the answer and stop; otherwise, proceed to step 4.
4. decrease the value of `t` by 1. Go to step 2