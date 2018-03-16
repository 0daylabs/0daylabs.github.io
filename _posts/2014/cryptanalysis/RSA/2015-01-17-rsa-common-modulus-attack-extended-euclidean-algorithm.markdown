---
author: AnirudhAnand
comments: true
date: 2015-01-17 16:41:50+00:00
layout: post
slug: rsa-common-modulus-attack-extended-euclidean-algorithm
title: 'RSA: Common Modulus attack with extended Euclidean algorithm'
wordpress_id: 5848
categories:
- rsa
summary: RSA common modulus attack using extended euclidean
---

RSA, a commonly used public key cryptosystem, is very secure if you use sufficiently large numbers for encryption. Even then there are attacks against it. If you are not already familiar with [RSA encryption mechanism](http://en.wikipedia.org/wiki/RSA_(cryptosystem)), I suggest you read more about it before continuing with this article.

Consider a scenario where a person encrypts same plain text, 2 different times, which he sends to 2 different people. Suppose you eavesdropped on the communication and got both the cipher texts (c1, c2) and the exponents(e1, e2) he used. You already know his Modulus N which is public. So is there a way you can decipher this ? Well the answer is yes.

In order to decrypt it, we use an algorithm called **extended euclidean** which makes our tasks much easier. But another condition we need to decrypt this is that the `gcd(e1, e2) = 1`

You must be thinking that why the hell did I just say gcd(e1, e2) should be equal to 1 ? Well here is the answer:

If you already know how the RSA encryption works, then by now you would have understood that

`c1 = m^e1 % N` and

`c2 = m^e2 % N`

Assume that we find out _**a**_ and _**b**_ such that `(e1 * a) + (e2 * b) = 1` then we can decode the plain text as `(c1 ^ a) + (c2 ^ b)`. If we substitute how **c1** and **c2** is calculated to above equation, we can get `m^(e1 * a + e2 * b) = m^1 = m`

Let us see how the algorithm works

`(e1*a) + (e2*b) = gcd(e1, e2)`

where e is exponents.

Now, we need to find **_a_** and** _b_**. In order to find _a_, since _gcd(e1, e2) = 1_, then we can say that _a_ is the **modular multiplicative inverse** of el and e2. After finding a, we can substitute the value to the above equation so that we can successfully obtain **b**. After getting a and b, we can get back the plain text by applying the equation

`plain = (c1^a) * (c2^b) %N`

Another issue that can happen here is that, in most of the cases, the value of **_b_** will be negative and hence it is difficult to apply in the above equation. So in order to simplify this, we find out another value named _**i**_, which is the modular multiplicative inverse of _c2_ w.r.t _N_. Then we can say `i^-b = c2^b`.

So now the equation becomes `plain = (c1^a) * (i^-b) %N`. This is what we need in order to decrypt the plain text. Let us look into a python script that can automate the above process:

 {% highlight python lineos %}
   #!/usr/bin/python3.4
   # Written by Anirudh Anand (lucif3r) : email - anirudh@anirudhanand.com   
   # This program will help to decrypt cipher text to plain text if you have
   # more than 1 cipher text encrypted with same Modulus (N) but different
   # exponents. We use extended Euclideangm Algorithm to achieve this.
   
   __author__ = 'lucif3r'
   
   import gmpy2
   
   
   class RSAModuli:
       def __init__(self):
           self.a = 0
           self.b = 0
           self.m = 0
           self.i = 0
       def gcd(self, num1, num2):
           """
           This function os used to find the GCD of 2 numbers.
           :param num1:
           :param num2:
           :return:
           """
           if num1 < num2:
               num1, num2 = num2, num1
           while num2 != 0:
               num1, num2 = num2, num1 % num2
           return num1
       def extended_euclidean(self, e1, e2):
           """
           The value a is the modular multiplicative inverse of e1 and e2.
           b is calculated from the eqn: (e1*a) + (e2*b) = gcd(e1, e2)
           :param e1: exponent 1
           :param e2: exponent 2
           """
           self.a = gmpy2.invert(e1, e2)
           self.b = (float(self.gcd(e1, e2)-(self.a*e1)))/float(e2)
       def modular_inverse(self, c1, c2, N):
           """
           i is the modular multiplicative inverse of c2 and N.
           i^-b is equal to c2^b. So if the value of b is -ve, we
           have to find out i and then do i^-b.
           Final plain text is given by m = (c1^a) * (i^-b) %N
           :param c1: cipher text 1
           :param c2: cipher text 2
           :param N: Modulus
           """
           i = gmpy2.invert(c2, N)
           mx = pow(c1, self.a, N)
           my = pow(i, int(-self.b), N)
           self.m= mx * my % N
          def print_value(self):
           print("Plain Text: ", self.m)
   
   
   def main():
       c = RSAModuli()
       N  =
       c1 =
       c2 =
       e1 =
       e2 =
       c.extended_euclidean(e1, e2)
       c.modular_inverse(c1, c2, N)
       c.print_value()
    
   if __name__ == '__main__':
       main()

   {% endhighlight %}

So here we deal with 2 function namely extended_euclidean and modular_inverse. The extended_euclidean function will help us in calculating the value of _**a**_ and _**b**_. In almost all the cases, the value of b will be negative and because of that, we will find out **i** in modular_inverse function and also calculate the plain text from the above explained equation.

NOTE:
	
  * In order to successfully launch an attack, the gcd of e1 and e2 must be equal to 1 or else this may not work.
	
  * The python script makes use of a library called gmpy2. If you don't have it already installed on your system, then do this:
  `sudo apt-get install python3-gmpy2`
	
  * When you change the exponents e1, e2 for encrypting the messages it also changes the Private/Public Key Pair. In practice we use the same key pairs so there wont be question of e1/e2 that makes decryption of crypt-text difficult.

If you have any issues related to the above tutorial or if you have a better way of scripting, please don't hesitate to comment below. We would like to hear from you :)