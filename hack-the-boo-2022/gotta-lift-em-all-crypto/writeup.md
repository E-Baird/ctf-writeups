# Gonna Lift 'Em All
Hack the Boo 2022

## Description

```
Quick, there's a new custom Pokemon in the bush called "The Custom Pokemon". Can you find out what its weakness is and capture it?
```

## Challenge Explanation

For this challenge, we get two ciphertexts as well as a bunch of the initial values that were used in the key generation. We are also given the source code that encrypted the ciphertext.

The flag is not actually the plaintext! Rather, the flag was used as part of the encryption process itself. So we will probably have to work our way through the way the encryption was done, and see if we can undo it.

## The Vulnerability

The source code includes a utility function to generate parameters. The use of `random.randint` is definitely not cryptographically secure, and it looks like it's used to generate `g` and `x`. Maybe this could be something?
 

```python
def gen_params():
  p = getPrime(1024)
  g = random.randint(2, p-2)
  x = random.randint(2, p-2)
  h = pow(g, x, p)
  return (p, g, h), x
```

We also get a function to encrypt:

```python
def encrypt(pubkey):
  p, g, h = pubkey
  m = bytes_to_long(FLAG)
  y = random.randint(2, p-2)
  s = pow(h, y, p)
  return (g * y % p, m * s % p)
```

This function has a major flaw. Lots of crypto is based around the use of functions that can't easily be inverted. In other words, math that's easy to do in one direction, but difficult to undo. One very common example of this is the discrete logarithm problem.

It is trivially easy to calculate the value of `x ^ y` modulo some value `p`. However, it is very difficult to undo this function. Even if you know `x` and `p`, there is no good, efficient solution to take a value for `x ^ y mod p` and work your way backward to retrieve the value of `y`. This is valuable for a crypto system, because you can generate values easily using `x ^ y mod p`, but your original value `y` can't be retrieved by an attacker.

However, the critical issue here is that this encryption function is *not* using discrete logs! It's running `g * y mod p`, not `g ^ y mod p`. Even though discrete logs are difficult to calculate, the same is not true for multiplication. There's nothing here preventing us from finding the multiplicative inverse of `g` and using it to retrieve the value of `y`.

## The Solution
We are given the values for `p`, `g`, `h`, as well as a keypair `(c1, c2)`. The `encrypt` function uses the flag as `m`, so our goal is to work backward to retrieve `m`. All of these operations are done congruent to `p`.

As a reminder, here's the encryption function again:
```python
def encrypt(pubkey):
  p, g, h = pubkey
  m = bytes_to_long(FLAG)
  y = random.randint(2, p-2)
  s = pow(h, y, p)
  return (g * y % p, m * s % p)
```

First, we'll use `p`, `g`, and `c1` to get `y`:
```
c1 = g * y mod p
y = g^-1 * g * y mod p      # (g^-1 * g) will cancel out
y = g^-1 * c1 mod p
```

Now that we have `y`, we have all the values needed to generate `s`:
```
s = h ^ y mod p
```

Finally, with `s` and `c2`, we can retrieve `m` using the same steps we did to find `y`:
```
c2 = s * m mod p
m = s^-1 * s * m mod p      # (s^-1 * s) will cancel out
m = s^-1 * c1 mod p
```

Here is the complete solution, in python:

```python
p = 163096280281091423983210248406915712517889481034858950909290409636473708049935881617682030048346215988640991054059665720267702269812372029514413149200077540372286640767440712609200928109053348791072129620291461211782445376287196340880230151621619967077864403170491990385250500736122995129377670743204192511487
g = 90013867415033815546788865683138787340981114779795027049849106735163065530238112558925433950669257882773719245540328122774485318132233380232659378189294454934415433502907419484904868579770055146403383222584313613545633012035801235443658074554570316320175379613006002500159040573384221472749392328180810282909
h = 36126929766421201592898598390796462047092189488294899467611358820068759559145016809953567417997852926385712060056759236355651329519671229503584054092862591820977252929713375230785797177168714290835111838057125364932429350418633983021165325131930984126892231131770259051468531005183584452954169653119524751729
(c1, c2) = (159888401067473505158228981260048538206997685715926404215585294103028971525122709370069002987651820789915955483297339998284909198539884370216675928669717336010990834572641551913464452325312178797916891874885912285079465823124506696494765212303264868663818171793272450116611177713890102083844049242593904824396, 119922107693874734193003422004373653093552019951764644568950336416836757753914623024010126542723403161511430245803749782677240741425557896253881748212849840746908130439957915793292025688133503007044034712413879714604088691748282035315237472061427142978538459398404960344186573668737856258157623070654311038584)

# invert g
g_inv = pow(g, -1, p)

# get c
y = c1 * g_inv % p      # g * g^-1 will cancel out

# get s
s = pow(h, y, p)

# invert s
s_inv = pow(s, -1, p)   # s * s^-1 will cancel out

# get m
m = c2 * s_inv % p

# get flagzz
print(hex(m))
```

When we run it, we get a hex value:
```
0x4854427b62335f6334723366756c5f7768336e5f316d706c336d336e37316e365f63727970373035793537336d355f316e5f3768335f6d756c3731706c316334373176335f36723075707d
```

Convert it to ascii, and we get our flag!
```
HTB{b3_c4r3ful_wh3n_1mpl3m3n71n6_cryp705y573m5_1n_7h3_mul71pl1c471v3_6r0up}
```


Success!