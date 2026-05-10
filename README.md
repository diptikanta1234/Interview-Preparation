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
    

        
        
        
