# Interview-Preparation
======================================
Q. WAP to display it as a prime number or not.

num = int(input('Enter a number: '))

#Defining a variable x as boolean so that it can be used later.
x = False

if num == 0 or num == 1:
    x=True
elif num > 1:
    for i in range(2, num):
        if (num%i) == 0:
            x = True;
            break;
            
if x == True:
    #print ( num, "is not a prime number." )
    print (f"{num} is not a prime number." )
else:
    print ( num, "is a prime number." )
    

    

        
        
        
