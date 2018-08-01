# Timisoara-CTF-Finals-2018-Crypto-Write-up
## Write-up for tasks Recap(250p) and Recess(350p)

This was a 2 in one problem. We were given a copy of the code that was running on a netcat in which the flags were in some files flag1,respectively flag2:<br>
```python
import signal
import sys
import os
import binascii
import random

os.chdir(os.path.dirname(os.path.abspath(__file__)))

MAGIC_NUMBER = 11
MSG_LENGTH = 119
CERT_CNT = MAGIC_NUMBER
coeffs = [random.randint(0, MAGIC_NUMBER**128) for i in range(MAGIC_NUMBER)]

def enc_func(msg):
    global coeffs
    msg = msg * 0x100 + 0xFF
    acc = 0
    cur = 1
    for coeff in coeffs:
        acc = (acc + coeff * cur) % (MAGIC_NUMBER**128)
        cur = (cur * msg) % (MAGIC_NUMBER**128)
    return acc % (MAGIC_NUMBER**128)

def get_hex_msg():
    try:
        msg = raw_input()
        msg = int(msg,16)
        return msg % (MAGIC_NUMBER ** (128) )
    except:
        print "Bad input"
        exit()

def encryption():
    print 'Input your message:'
    msg = get_hex_msg()
    ct = enc_func( msg )
    print "Encryption >>>> %#x" %  ct

def challenge1():
    for i in range(CERT_CNT):
        msg = random.randint(0, MAGIC_NUMBER**(MSG_LENGTH) )
        ct = enc_func( msg )
        print 'Encrypt this: %#x' % msg
        ct2 = get_hex_msg()
        if ct != ct2:
            print "Your input %#x should have been %#x" % (ct2, ct)
            exit()
    print "You win challenge 1"
    print open("flag1").read()

def challenge2():
    for i in range(CERT_CNT):
        msg = random.randint(0, MAGIC_NUMBER**(MSG_LENGTH))
        ct = enc_func( msg )
        print 'Decrypt this: %#x' % (ct)
        msg2 = get_hex_msg()
        if enc_func(msg) != enc_func(msg2):
            print "Your input %#x should have been %#x" % (msg2, msg)
            exit()
    print "You win challenge 2"
    print open("flag2").read()

def input_int(prompt):
    sys.stdout.write(prompt)
    try:
        n = int(raw_input())
        return n
    except ValueError:
        return 0
    except:
        exit()

def menu():
    while True:
        print "Horrible Crypto"
        print "1. Arbitrary Encryption"
        print "2. Encryption Challenge"
        print "3. Decryption Challenge"
        print "4. Exit"
        choice = input_int("Command: ")
        {
                1: encryption,
                2: challenge1,
                3: challenge2,
                4: exit,
        }.get(choice, lambda *args:1)()

if __name__ == "__main__":
    signal.alarm(15)
   a menu()
```
Ok, so let's analyse what we are given and what we have to do:<br>
Running the code we arrive to a menu:<br>
> Horrible Crypto
> 1. Arbitrary Encryption
> 2. Encryption Challenge
> 3. Decryption Challenge
> 4. Exit

So, we are given an encryption oracle and two challanges, encryption(**task recap**) and decryption(**task recess**)<br>
### Understanding the encryption
Let's start with the encryption function:<br>
```python
def enc_func(msg):
    global coeffs
    msg = msg * 0x100 + 0xFF
    acc = 0
    cur = 1
    for coeff in coeffs:
        acc = (acc + coeff * cur) % (MAGIC_NUMBER**128)
        cur = (cur * msg) % (MAGIC_NUMBER**128)
    return acc % (MAGIC_NUMBER**128)

def get_hex_msg():
    try:
        msg = raw_input()
        msg = int(msg,16)
        return msg % (MAGIC_NUMBER ** (128) )
    except:
        print "Bad input"
        exit()

def encryption():
    print 'Input your message:'
    msg = get_hex_msg()
    ct = enc_func( msg )
    print "Encryption >>>> %#x" %  ct
```
Inside `encryption()` the input is a hex number which gets decoded to an int and then actually encrypted in `enc_func(msg)`<br>
Now, let's focus on how the encryption works:<br>
It takes the message, it multiplies it by 256 adds 255 `msg = msg * 0x100 + 0xFF` and then computes the polynomial `coeffs(msg)` modulo 11<sup>128</sup> (`MAGIC_NUMBER = 11` is a constant), and that's our ciphertext.<br>
### Task Recap
Recap was the first challenge. The adversary is given 11 messages to encrypt, by succesfully encrypting the messages, the flag is given to the adversary.<br>
```python
def challenge1():
    for i in range(CERT_CNT):
        msg = random.randint(0, MAGIC_NUMBER**(MSG_LENGTH) )
        ct = enc_func( msg )
        print 'Encrypt this: %#x' % msg
        ct2 = get_hex_msg()
        if ct != ct2:
            print "Your input %#x should have been %#x" % (ct2, ct)
            exit()
    print "You win challenge 1"
    print open("flag1").read()
```
so basically we need to find the coefficients of the polynomial **coeffs** in order to be able to encrypt. Note that we have an encryption oracle so practically we can find any coefficient of the polynomial **coeffs** right? Well that's great because we can then recreate the original polynomial by creating a **Lagrange polynomial** of degree equal to the degree of the polynomial **coeffs** <br>
You can read more about it on [Wikipedia](https://en.wikipedia.org/wiki/Lagrange_polynomial), the math is pretty simple, and there's an image that visually describes very well the algorithm.
Here's the solution for the problem:<br>
**NOTE**: since the contest's servers don't work anymore you will have to run the challange code locally<br>
```python
from pwn import *
from Crypto.Util.number import inverse

MOD=11**128

def transform(x):
	return x*256+255

#Polynomial part
class Polynomial(list):

	def __init__(self,coeffs):
		self.coeffs = coeffs

	def evaluate(self,x):
		val = 0
		for i in range(len(self.coeffs)):
			val = (val + x**i * self.coeffs[i]) % MOD
		return val

	def raise_degree(self,x):
		coeffs=[]

		for i in range(x):
			coeffs.append(0)

		for i in range(len(self.coeffs)):
			coeffs.append(self.coeffs[i])
		
		self.coeffs=coeffs
	
	def add_to_degree(self,x,y):
		while(len(self.coeffs)<=x):
			self.coeffs.append(0)
		
		self.coeffs[x]=(self.coeffs[x]+y)%MOD

	def add_poly(self,x):
		while(len(self.coeffs)<len(x.coeffs)):
			self.coeffs.append(0)
		
		for i in range(len(x.coeffs)):
			self.coeffs[i]=(self.coeffs[i]+x.coeffs[i])%MOD

	def multiply(self,x):
		for i in range(len(self.coeffs)):
			self.coeffs[i]=(self.coeffs[i] * x)%MOD

	def multiply_with_poly(self,p):
		coeffs=Polynomial([])		

		for i in range(len(self.coeffs)):
			q=Polynomial(p.coeffs)
			q.raise_degree(i)
			q.multiply(self.coeffs[i])
			coeffs.add_poly(q)
		self.coeffs=coeffs.coeffs

	def print_poly(self):
		return self.coeffs

#Lagrange Interpolation part
def Lagrange_Basis_Polynomial(xlist,index):
	l=Polynomial([1])
	for i in range(len(xlist)):
		if(i==index):
			continue
		p=Polynomial([(MOD-xlist[i])%MOD,1])
		p.multiply(inverse((MOD+xlist[index]-xlist[i])%MOD,MOD))
		l.multiply_with_poly(p)
	return l

def Lagrange_Polynomial(xylist):
	xlist=[]
	ylist=[]
	L=Polynomial([])

	for a in xylist:
		xlist.append(a[0])
		ylist.append(a[1])

	for i in range(len(ylist)):
		l=Lagrange_Basis_Polynomial(xlist,i)
		l.multiply(ylist[i])
		L.add_poly(l)
	return L

#Main part
xylist=[]
s=remote('localhost',1337)

for i in range(11):
	s.recvuntil('Command: ')
	s.sendline('1')
	s.recvuntil('Input your message:')	
	s.sendline(hex(i)[2:])
	s.recvline()
	x=s.recvline()[18:][:-1]
	x=int(x,16)
	xylist.append((transform(i),x))

print "Values of the polynomial:"

for x in xylist:
	print 'f('+str(x[0])+') = '+str(x[1])

coeffs=Lagrange_Polynomial(xylist)

print "\nReconstructed polynomial"

for i,x in zip([i for i in range(11)],coeffs.print_poly()):
	print str(i)+':',x

print '\nStarting challenge 1!'

s.recvuntil('Command: ')
s.sendline('2')

for i in range(11):
	s.recvuntil('Encrypt this: ')
	x=int(s.recvline()[2:],16)
	s.sendline(str(hex(coeffs.evaluate(transform(x)))[2:]))
	print 'step',i+1,'done'

print '\n'+s.recvline()
```


### Task Recess

