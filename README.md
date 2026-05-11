# Interview-Preparation
======================================

## Question 1
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

## Question 2
**Write a Python program to display the Fibonacci series up to a given number of terms.**

---

## Python Program

```python
num = int(input("Enter the ending number: "))

# Initial values
n1, n2 = 0, 1

#using for loop
'''
for i in range(0, num):
    print(n1)

    nth = n1 + n2

    # Update values
    n1 = n2
    n2 = nth
'''

#we can use while loop as well.
count=0
while (count < num):
    print(n1)
    nth=n1+n2
    
    #update values
    n1=n2
    n2=nth
    
    count +=1

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
## Question 3
**Write a Python program to display the Factorial of a given number.**

---

## Python Program

```python
num=int(input("Enter a number for factorial : "))

fact = 1

if num == 0:
    print(f"factorial of {num} is 0")
elif num == 1:
    print( f"factorial of {num} is 1")
else:
    
    #using for loop
    '''
    for i in range(2,num+1):
        fact = i * fact
    print (f"Factorial of {num} is {fact}")
    '''
    
    #using while loop
    count=1
    while count <= num:
        fact = count * fact
        count +=1
    print(f"Factorial of {num} is {fact}.")

```
---
## Question 4
**Write a Python program to display its a leap year or not.**

---

## Python Program
'''
Logic: A a year is called a leap year -
 1. if its a century year( ending with 00 [ divisible by 100] ), it should be divisible by 400 
 2.  If its a non-century year, it should be divisible by 4.
 '''

```python
year = int(input("Enter a year : "))

# century year means it should be divisible by 100
if (year % 400 == 0) and (year % 100 == 0):
    print("{0} is a leap year".format(year))

# non-century year means it should not be divisible by 100
elif (year % 4 == 0) and (year % 100 != 0):
    print("{0} is a leap year".format(year))

else:
    print(f"{year} is not a leap year.")

```
---
## Question 5
**Write a Python program to reverse a string.**

---

## Python Program

```python
str = input("Enter a string : ")

rev_str = reversed(str)

print(type(rev_str)) #<class 'reversed'>
print(type(str))     #<class 'str'>

#prints the reversed string in list. 
#print(list(rev_str))

print(''.join(list(rev_str)))

'''
join() is a string method used to combine multiple items into a single string.
'separator'.join(iterable)

letters = ['P', 'y', 't', 'h', 'o', 'n']
print(''.join(letters)) // output: Python
print('*',join(letters)) // output: P*y*t*h*o*n
''' 
```
---    

## break Vs continue
**
break	-> Stops the entire loop and comes out of the loop.
continue ->	Skips only the current iteration and continues the loop
**

<img width="1149" height="214" alt="image" src="https://github.com/user-attachments/assets/2c2a7aba-dd7a-4f55-95a9-9e0ce0aecacc" />

<img width="1150" height="236" alt="image" src="https://github.com/user-attachments/assets/79e3fb96-2679-4677-85b8-37eccfa78c5b" />

---
        
        
        
        
