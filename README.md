# Interview-Preparation
======================================

## Question
**Q. WAP to display whether a number is a prime number or not.**

---

## Python Program

```python
num = int(input('Enter a number: '))

# Defining a variable x as boolean so that it can be used later.
x = False

if num == 0 or num == 1:
    x = True

elif num > 1:
    for i in range(2, num):
        if (num % i) == 0:
            x = True
            break

if x == True:
    print(f"{num} is not a prime number.")
else:
    print(f"{num} is a prime number.")
    #print(num, "is a prime number.")  --We can use this format also
```

---

## Sample Output

### Example 1

```text
Enter a number: 7
7 is a prime number.
```

### Example 2

```text
Enter a number: 10
10 is not a prime number.
```

---

## Explanation

A **prime number** is a number greater than `1` that has only two factors:

- `1`
- The number itself

### Logic Used

- `0` and `1` are not prime numbers.
- The program checks divisibility from `2` to `num - 1`.
- If the number is divisible by any value in that range, it is **not prime**.
- Otherwise, it is a **prime number**.

---

## Question
**Write a Python program to display the Fibonacci series up to a given number of terms.**

---

## Python Program

```python
num = int(input("Enter the ending number: "))

# Initial values
n1, n2 = 0, 1

for i in range(0, num):
    print(n1)

    nth = n1 + n2

    # Update values
    n1 = n2
    n2 = nth
```

---

## Sample Output

```text
Enter the ending number: 5
0
1
1
2
3
```

---

```text
lets take iteration umber as num=5
i=0 t0 4
i=0 --> n1=0,n2=1 nth=0+1=1  updated n1=1, n2=1
i=1 --> n1=1,n2=1 nth=1+1=2  updated n1=1, n2=2
i=2 --> n1=1,n2=2 nth=1+2=3  updated n1=2, n2=3
i=3 --> n1=2,n2=3 nth=2+3=5  updated n1=3, n2=5
i=4 --> n1=3,n2=5 nth=3+5=8  updated n1=5, n2=8
```

---
        
        
        
