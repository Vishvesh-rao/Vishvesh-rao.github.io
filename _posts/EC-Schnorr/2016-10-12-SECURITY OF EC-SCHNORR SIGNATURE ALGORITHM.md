---
title: SECURITY OF EC-SCHNORR SIGNATURE ALGORITHM
date:  2022-01-22 12:30:06
author: Vishvesh
author_url: https://twitter.com/The_Str1d3r
mathjax: true
categories:
  - Crypto
tags:
  - "Bitcoin protocols"
  - "Schnorr"
  - "Signaturealgirithm"

url : demo.hedgedoc.org/gK9iEJ0tQka_BrOt1IGSyA?both
---
<html>
  <head>
    <script type="text/javascript" async
  src="https://cdnjs.cloudflare.com/ajax/libs/mathjax/2.7.7/MathJax.js?config=TeX-MML-AM_CHTML">
    </script>


<script type="text/javascript" src="https://cdnjs.cloudflare.com/ajax/libs/mathjax/2.7.1/MathJax.js?config=TeX-AMS_HTML">
  MathJax.Hub.Config({
    "HTML-CSS": {
      availableFonts: ["TeX"],
    },
    tex2jax: {
      inlineMath: [['$','$'],["\\(","\\)"]]},
      displayMath: [ ['$$','$$'], ['\[','\]'] ],
    TeX: {
      extensions: ["AMSmath.js", "AMSsymbols.js", "color.js"],
      equationNumbers: {
        autoNumber: "AMS"
      }
    },
    showProcessingMessages: false,
    messageStyle: "none",
    imageFont: null,
    "AssistiveMML": { disabled: true }
  });
</script>
</head>
</html>

Recently Elliot ( a.k.a [@robot__dreams](https://twitter.com/robot__dreams) ) published some crypto/bitcoin related challenges on the EC-schnorr algorithm and key sharing algorithm used to secure bitcoin wallets. 

This blog post is my detailed explanation of my approach in solving all the challenges.

We will start with some basics about the EC-schnorr algortithm and its relevance in blockchain and more specifically in bitcoin.

## So first up why EC-schnorr?
Whats the relevance of this algorithm in bitcoin? well all cryptocurrencies have a basic feature and that is transactions using acccount addresses. To verify that a certain transaction came from the correct account digital signatures are used i.e they validate a transaction. The private key is used to sign a certain message and the other nodes can apply the public key to get back the message from the signature and verify that with the attached message, if it doesnt verify that transaction is rejected and not registered on the ledger.

Currently bitcoin uses the ECDSA algorithm for achiving this. But now in the recent *Bitcoin Improvement Proposal [BIP0340](https://github.com/bitcoin/bips/blob/master/bip-0340.mediawiki)* EC-Schnorr algorithm has been proposed to replace the less efficient ECDSA system since its simpler and more faster.

## Primer on EC-Schnorr

It is an easy to understand algorithm and simple to implement and verify compared to its counterparts.

This like its predecessor, ECDSA operates over the secp256k1 curve paired with sha256 hashes for authentication.


Following is the signing /verification method as used in the algorithm



### Signing

$$m  \qquad\quad\quad\; :Message \qquad\qquad\qquad\;\;\; (byte \; array)$$

$$k \qquad\qquad\;\;\; :Private \; Key \quad\quad\quad\quad\quad  (integer)$$

$$G \qquad\qquad\;\; :Generator \; Point \quad\quad\;\;\;\;\; (curve point)$$

$$P = k*G \quad :Public \; Key \quad\qquad\quad\;\;\;\;\;\;\; (curve point)$$

$$z \quad\quad\qquad\;\;\;\; :Private \; nonce \qquad\qquad\; (integer)$$

$$R = z*G  \quad\; :Public \;nonce \qquad\qquad\;\;\; (curve point)$$

$$e = Hash(r||P||m) \quad :aggregated hash \quad \; (integer)$$

<br>

> $Here \; r,P,m \;\; are \; all \; represented \; in \;  bytes$
> $And \; r \; is \; the \; x \; cordinate \; of \; point \; R$

<br>

$$s = z + Hash(r||P||m)*k \quad :signature \quad (integer)$$

$$(r, s) \qquad\quad\;\;\;\; :Signature \quad\qquad\qquad\;\;  (integer, integer)$$

<br>

### Verification

**Public Values**

$$(r, s) \qquad\quad\;\;\;\; :Signature$$

$$P \quad\qquad\qquad\; :Public \; Key$$

$$m \quad\qquad\qquad :Message$$

**Calculated Values**

$$R \quad\qquad :Public \; nonce \quad (calculated \; from \; r)$$

$$e \quad\qquad\ :Hash(r||P||m)$$

**Verifying**

$$s = z + Hash(r||P||m)*k$$

$$s*G = z*G + Hash(r||P||m)*k*G$$

$$s*G => R + Hash(r||P||m)*P \tag{if true then verified}$$

Now that we know the basics of the signing/verification process we can move on the challenges.....

## Challenges

1. **[EC-Schnorr Nonce Reuse](https://vishvesh-rao.github.io/posts/SECURITY-OF-EC-SCHNORR-SIGNATURE-ALGORITHM/#ec-schnorr-nonce-reuse)**
2. **[Linearly Related Nonces](https://vishvesh-rao.github.io/posts/SECURITY-OF-EC-SCHNORR-SIGNATURE-ALGORITHM/#linearly-related-nonces)**
3. **[Multisignature Forgery](https://vishvesh-rao.github.io/posts/SECURITY-OF-EC-SCHNORR-SIGNATURE-ALGORITHM/#multisignature-forgery)**
4. **[Wallet Key Reconstruction](https://vishvesh-rao.github.io/posts/SECURITY-OF-EC-SCHNORR-SIGNATURE-ALGORITHM/#wallet-key-reconstruction)**

## EC-Schnorr Nonce Reuse

### CHALLENGE

We are given the following values. Here the same private key has been used for signing two different messages, the aim is to extract the private key

``` 
You're given Schnorr signatures on two different messages signed by the same 
private key. Fortunately for you (the adversary), the signer screwed up their
implementation of BIP-340 and reused a nonce. Can you capitalize on this fatal 
error and extract the signer's private key?

Note: You may find it helpful to interpret some of the byte strings as ASCII,
in order to check your work.

Public Key: 463F9E1F3808CEDF5BB282427ECD1BFE8FC759BC6F65A42C90AA197EFC6F9F26
Message 1: 6368616E63656C6C6F72206F6E20746865206272696E6B206F66207365636F6E
Signature 1: F3F148DBF94B1BCAEE1896306141F319729DCCA9451617D4B529EB22C2FB521A32A1DB8D2669A00AFE7BE97AF8C355CCF2B49B9938B9E451A5C231A45993D920
Message 2: 6974206D69676874206D616B652073656E7365206A75737420746F2067657420
Signature 2: F3F148DBF94B1BCAEE1896306141F319729DCCA9451617D4B529EB22C2FB521A974240A9A9403996CA01A06A3BC8F0D7B71D87FB510E897FF3EC5BF347E5C5C1
```
### SOLUTION
so here the public key is given along with the two signatures for the two different messags 

***known Values:***

$$n \quad\;\;\;\; :Curve \; Order$$

$$G \quad\;\;\; :Generator \; for \; secp256k1 \; curve$$

$$P \quad\;\;\; :Public \; Key$$

$$m_1 \quad :Message \; 1$$

$$m_2 \quad :Message \; 2$$

$$(r,s_1) \quad :S_1 \; Signature  \; of \; M_1$$

$$(r,s_2) \quad :S_2 \; Signature  \; of \; M_2$$

<br>

> $\; r \; is \; the \; x \; cordinate \; of \; Public \; Nonce \; R$

<br>

***Calculated Values:***

$$R \quad\; :Public \; nonce \quad (calculated \; from \; r)$$

$$e_1 \quad :Hash(r||P||m_1)$$

$$e_2 \quad :Hash(r||P||m_2)$$

<br>

Now that we have all the values we can write the equations for both $s_1$ and $s_2$

$$s_1 = z + Hash(r||P||m_1)*k  \tag{1}$$

$$s_2 = z + Hash(r||P||m_2)*k  \tag{2}$$

Our aim is to get $$k$$

since the $z$ ( private nonce ) is same in both (1) and (2) we can do $$s_1 - s_2 \tag{3}$$

so we will get:

$$z + Hash(r||P||m_1)*k - (z + Hash(r||P||m_2)*k)$$

$$z + Hash(r||P||m_1)*k - z - Hash(r||P||m_2)*k$$

$$Hash(r||P||m_1)*k - Hash(r||P||m_2)*k$$

$$ e_1*k - e_2*k $$

<br>

$$(e_1-e_2)*k = s_1 - s_2 \tag{from 3}$$
<br>

$$k = \frac{s_1 - s_2}{e_1-e_2} $$

Since all these operations are done in a finite field final equation will be

$$k \equiv (s_1 - s_2)*(e_1-e_2)^{-1} \bmod n \tag{4}$$

All the values in the RHS of (4) are known to us thus we can easily get the Private Key $k$

### IMPLEMENTATION
```python
from Crypto.Util.number import long_to_bytes, bytes_to_long, inverse
import hashlib

n = 0xFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFEBAAEDCE6AF48A03BBFD25E8CD0364141

p = "463F9E1F3808CEDF5BB282427ECD1BFE8FC759BC6F65A42C90AA197EFC6F9F26"
m1 = "6368616E63656C6C6F72206F6E20746865206272696E6B206F66207365636F6E"
m2 = "6974206D69676874206D616B652073656E7365206A75737420746F2067657420"
s1 = "F3F148DBF94B1BCAEE1896306141F319729DCCA9451617D4B529EB22C2FB521A32A1DB8D2669A00AFE7BE97AF8C355CCF2B49B9938B9E451A5C231A45993D920"
s2 = "F3F148DBF94B1BCAEE1896306141F319729DCCA9451617D4B529EB22C2FB521A974240A9A9403996CA01A06A3BC8F0D7B71D87FB510E897FF3EC5BF347E5C5C1"

p = bytes.fromhex(p)

s1 = bytes.fromhex(s1)
s2 = bytes.fromhex(s2)

r1,s1 = s1[0:32],s1[32:]
r2,s2 = s2[0:32],s2[32:]

r =r1

m1 = bytes.fromhex(m1)
m2 = bytes.fromhex(m2)

s1 = bytes_to_long(s1)
s2 = bytes_to_long(s2)

def int_from_bytes(b: bytes) -> int:
    return int.from_bytes(b, byteorder="big")

def tagged_hash(tag: str, msg: bytes) -> bytes:
    tag_hash = hashlib.sha256(tag.encode()).digest()
    return hashlib.sha256(tag_hash + tag_hash + msg).digest()

e1 = int_from_bytes(tagged_hash("BIP0340/challenge", r + p + m1)) % n
e2 = int_from_bytes(tagged_hash("BIP0340/challenge", r + p + m2)) % n

secret_key = ((s2-s1)*(inverse(e2-e1,n)))%n

print(long_to_bytes(secret_key))

secret_key = hashlib.sha256(long_to_bytes(secret_key)).hexdigest()
```

> Here `s1` and `s2` represent the combined 64 byte signature $(r,s)$ of which 32 bytes are of $r$ and 32 bytes of $s$ 

The secret key comes out to be `b'congratulations you found the sk'`.

### MITIGATION

So now that we have exploited the vulnerability how do we mitigate it well in this case its as simple as just using a different random private nonce commitment each time you sign a meassage.

## Linearly Related Nonces
### CHALLENGE

The poor signer realized their mistake and upgraded their implementation to randomly generate (private) nonces. Unfortunately, they also didn't get the memo to use a secure PRNG, and ended up using a Linear Congruential Generator instead.

You're given Schnorr signatures on two different messages signed by the same private key. Although the signatures both verify under BIP-340, the two private nonces are related via $z2 = a * z1 + b$ , where $a = 31337$ and $b = 69420$.

Can you still extract the signer's private key?

```
Public Key
21922E7D5988A711123794D70B19C2827B1630BC2AB99887418D9EF4AFDB1AC2

Message 1
49276D20626574746572207769746820636F6465207468616E20776974682077

Signature 1
19D6493FBA397CDD1C1E10F9AB51E65531D587D7C53C04673779E1A307AC795CF801B1BF3D103771F74C5F70BB3A3557D87E5116294A9ABD357DC4367D123C9D

Message 2
4265696E67206F70656E20736F75726365206D65616E7320616E796F6E652063

Signature 2
0293422DCE97000231B98AFE3CBE405601D4129296AB902822514DF9B2F0BC9D7FC2B9C64FA080688D020407900CE9DE887B9CBB25C34280DAB6E172CC39C2F0
```
### SOLUTION

Here instead of reused nonce we are given a relation between the two nonces represented by $z_2 = 31337 * z_1 + 69420$

$z_1$ and $z_2$ are the private nonce value for the two 64 byte signatures $S_1$ and $S_2$

Since we already have an equaiton with the two nonces the obvious next step would be to get another equation in $z_1$ and $z_2$ and solving for both.

To do that lets look at the provided data:

***known Values:***

$$n \quad\;\;\;\; :Curve \; Order$$

$$G \quad\;\;\; :Generator \; for \; secp256k1 \; curve$$

$$P \quad\;\;\; :Public \; Key$$

$$m_1 \quad :Message \; 1$$

$$m_2 \quad :Message \; 2$$

$$(r_1,s_1) \quad :S_1 \; Signature  \; of \; M_1$$

$$(r_2,s_2) \quad :S_2 \; Signature  \; of \; M_2$$

> $\; r_1 \; and \; r_2 \; are \; the \; x \; cordinates \; of \; their \; corresponding \; Public \; Nonce \; values$


***Calculated Values:***

$$e_1 \quad :Hash(r_1||P||m_1)$$

$$e_2 \quad :Hash(r_2||P||m_2)$$

Now that we have all the values we can write the equations for both $s_1$ and $s_2$

$$s_1 = z_1 + Hash(r_1||P||m_1)*k  \tag{1}$$

$$s_2 = z_2 + Hash(r_2||P||m_2)*k  \tag{2}$$

Our aim is to get an equation in $z_1$ and $z_2$ for that we wil need to eliminate $k$ from $(1)$ and $(2)$
<br>
$$ Taking \;\; (1) :$$
<br>

$$s_1 - z_1 = Hash(r_1||P||m_1)*k $$ 

$$s_1 - z_1 = e_1*k $$ 

$$ k \equiv (s_1 - z_1)*e_1^{-1} \bmod n  \tag{3}$$ 
<br>
$$ Taking \;\; (2) :$$
<br>

$$s_2 - z_2 = Hash(r_2||P||m_2)*k $$ 

$$s_2 - z_2 = e_2*k $$ 

$$ k \equiv (s_2 - z_2)*e_2^{-1} \bmod n  \tag{4}$$ 
<br>
$$From \;\; (3) \;\; and \;\; (4)$$
<br>

$$ (s_1 - z_1)*e_1^{-1} \equiv (s_2 - z_2)*e_2^{-1} \bmod n$$ 

$$ s_1*e_1^{-1} - z_1*e_1^{-1} \equiv s_2*e_2^{-1} - z_2*e_2^{-1} \bmod n $$

$$ z_2*e_2^{-1}  - z_1*e_1^{-1} \equiv s_2*e_2^{-1} - s_1*e_1^{-1}\bmod n $$

<br>

let $$s_2*e_2^{-1}-s_1*e_1^{-1}$$ be $c$ so we get:
$$z_2*e_2^{-1} - z_1*e_1^{-1} \equiv c\bmod\ n \tag{5}$$

$$z_2 - a * z_1 = b \tag{6}$$
from $(5)$ and $(6)$ we can get the values of both nonces and then using either $(3)$ or $(4)$ we can get the private key $k$.

### MITIGATION

- Again the answer here is simple read the definiton of nonce :). The word nonce itself is a combination of two words ***Only Once***. Use a particular nonce only once in the whole algorithm implementation.

- If you still dint understand here the definition from wikipedia `nonce (number once) is an arbitrary number that can be used just once in a cryptographic communication`. 

- Lastly, use a secure CSPRNG and dont go diving into implementing your own techniques for nonce generation.

## Multisignature Forgery

This challenge was a bit on the harder side compared to the other two in the sense it required a bit more in dept knowledge of Elliptic Curves and EC-Schnorr signatures.

So before we actually dive into the challenge lets do a quick brief on multisignatures in EC-Schnorr. 

### EC-Schnorr Multisignatures

- Multisignature is used when a certain transaction is to be signed by more than one node. I.e before that transaction is broadcasted onto to the blockchain it is signed by multiple users. 

- In EC-Schnorr this ***multisig*** feature is achived via a property it has, namely **Linearity**. Schnorr signatures are Linear.

- Simply meaning on adding two schnorr signatures the resultant is also a valid schnorr signature.

- This property is very important as it enables multiple parties to share a common signature instead of generating multiple signaters over a message and sending it over a transaction.

- The public keys of all signers are added and so are the public nonces R, and are translated into the public key and nonce commitment value for the aggregated multisig.  

Following are the steps to generate a two party multisignature: 
<br>

> **All the terms used are as explained in [this](https://vishvesh-rao.github.io/posts/SECURITY-OF-EC-SCHNORR-SIGNATURE-ALGORITHM/#primer-on-ec-schnorr) section**

<br>

#### ADDITIONAL TERMS:

$$ P_{agg} \quad :Aggregate \;of \;public \;keys \;(P_1+P_2) $$

$$ z_{agg} \quad :Aggregate \;of \;private \;nonces \;(z_1+z_2) $$

$$ r_{agg} \quad :x \;cordinate \;of z_{agg}*G $$


#### MULTI SIG  CREATION:

***Individual sig***

$$s_1 = z_1 +  Hash(r_{agg}||P_{agg}||m)*k_1 \quad :sig \;of \;person 1$$

$$s_1 = z_2 +  Hash(r_{agg}||P_{agg}||m)*k_2 \quad :sig \;of \;person 2$$

***Aggregated sig***

$$s = s_1+s_2$$

$$s = (z_1+z_2) +  Hash(r_{agg}||P_{agg}||m)*(k_1+k_2)$$

<br>

$$s = z_{agg} +  Hash(r_{agg}||P_{agg}||m)*k \tag{aggregated signature}$$

#### MULTI SIG VERIFICATION:

$$s*G = (z_1 + z_2)*G + Hash(r_{agg}||P_{agg}||m)*(k_1 + k_2)*G$$

$$s*G = z_{agg}*G + Hash(r_{agg}||P_{agg}||m)*(P_1 + P_2)$$

<br>

$$s*G = z_{agg}*G + Hash(r||P_{agg}||m)*P_{agg} \tag{if true then verified}$$

<br>

***Now lets move on to the actul challenge...***

### CHALLENGE

This challenge is a server based nc challenge but for ease of solving the goal is just to write a function which will generate a forged function and pass the schnorr verification fucntion.

![](https://codimd.s3.shivering-isles.com/demo/uploads/b701362d73f35ebdd3fa8d37f.png)

You can download the files from [here](https://gist.github.com/robot-dreams/6300fde4017eefcf02c241f203a75162)

This is the function we have to implement:

```python
def forge_signature(honest_signer, msg):
    """
    TODO: Your implementation here!

    Your goal is to return a tuple with two elements:
        - A list of public keys, at least one of which is the
          honest signer's public key
        - A valid BIP-340 signature for the input msg, when verified
          against the aggregate public key which is the sum of the
          individual public keys in the above list

    In trying to generate a forgery, you may interact with the
    honest signer by calling public methods as much as you want;
    however, to make this attack realistic, you're NOT allowed to:
        - Access private fields of the honest signer
        
        - Ask the honest signer to generate a partial signature
          on the same (pubkeys, msg) pair as the forgery you
          output

    Hopefully these restrictions are obvious and sensible; otherwise
    the challenge would be trivial.

    Good luck!
    """
```

### SOLUTION

So we have already seen how schnorr verification works for mutisignature now lets look at the custom implemetation of schnorr from `naive_multisig.py`

Analysing the class `NaiveMultisigSigner`


```python
class NaiveMultisigSigner:
    def __init__(self, seckey=None):
        if seckey is None:
            seckey = secrets.token_bytes(32)
        self.seckey = seckey
        self.pubkey = pubkey_gen(self.seckey)
        self.seen_queries = set()

    def get_pubkey(self):
        return self.pubkey

    def gen_partial_pubnonce(self):
        self.secnonce = secrets.token_bytes(32)
        return pubkey_gen(self.secnonce)

    def gen_partial_sig(self, pubkeys, aggnonce, msg):
        assert pubkey_gen(self.seckey) in pubkeys
        assert len(aggnonce) == 33
        assert len(msg) == 32

        X = xonly_point_agg(pubkeys)
        R = point_from_cbytes(aggnonce)
        r1 = xonly_int(self.secnonce, R)
        x1 = xonly_int(self.seckey, X)
        agg_pubkey = bytes_from_point(X)
        e = int_from_bytes(tagged_hash("BIP0340/c;hallenge", bytes_from_point(R) + agg_pubkey + msg)) % n

        self.seen_queries.add((agg_pubkey, msg))
        return bytes_from_int((r1 + e * x1) % n)
```

So we can see that for each class object created it is assigned with a public Key and a public nonce commitment value which can be acessed by anyone.

The `gen_partial_sig` takes 3 arguements with pubkeys and nonce as lists and the msg in bytes.
On line 28 the code: `self.seen_queries.add((agg_pubkey, msg))` this means an object can sign a particular message only once with its private key as its recorded.

Now looking at the `test_forgery()` function:

```python
def test_forgery():
    honest_signer = NaiveMultisigSigner()
    msg = b'send all of Bob\'s coins to Alice'

    pubkeys, sig = forge_signature(honest_signer, msg)
    agg_pubkey = bytes_from_point(xonly_point_agg(pubkeys))

    assert honest_signer.get_pubkey() in pubkeys
    assert (agg_pubkey, msg) not in honest_signer.seen_queries
    assert schnorr_verify(msg, agg_pubkey, sig)
    print("sucess!!")
```
So from the first 2 assert functions we can get that the list of public keys should include the honest signers key and honest signer cannot sign the msg so inside the `forge_signature` we need to create another class object which will forge the signature acessing only the public parameters of honest signer.

TO achive this we need the verification service to think the aggregate public key and nonce of the multisignature came from only one signer.

This is achieved via the `Rogue Key Attack`. This is an attack vector that can be exploited in such implementations of schnorr multisignature where the an attacker who is part of the group creating the multisignature could claim a false Public Key, and control the Multi-Sig.

The following steps describe the attack:

- `honest_signer` and `forged_signer` take part in a 2 party multisig.

$(k_h,P_h)\quad :honest\_signer \;key \;pair  \;(PrivateKey,PublicKey)$
$(k_f,P_f)\quad :forged\_signer \;key \;pair  \;(PrivateKey,PublicKey)$

$P_{agg} \quad :Aggregate \;of \;public \;keys \;(P_h+P_f) $

$R_{agg} \quad :Aggregate \;of \;private \;nonces \;(R_h+R_f) $

$r_{agg} \quad\; :x \;cordinate \;of R_{agg}$

- Now `forged_signer` can send a false public key $P_{false}$ to add to the list such that resulting aggregate public key is `forged_signer` public key.

$P_{false} = P_f - P_h$

- `forged_signer` broadcasts $P_{false}$ as his public key:

$P_{agg} = P_h + P_{false}$

$P_{agg} = P_h + (P_f - P_h)$

$P_{agg} = P_f$

- Since the aggregate key is now `forged_signer` true public key, `forged_signer` effectively controls the multi-sig.
- The same procedure is done to generate the public nonce $R_{agg}$

$R_{false} = R_f - R_h$

$P_{agg} = R_h + R_{false}$

$R_{agg} = R_h + (R_f - R_h)$

$R_{agg} = R_f$

- Now anyone any message signed with these parametrs is effictively signed only by `forged_signer`.

### IMPLEMENTATION

```python
def forge_signature(honest_signer, msg):
   
    forged_signer = NaiveMultisigSigner()
    X1 = honest_signer.get_pubkey()
    X2 = forged_signer.get_pubkey()

    X3 = b"NEG"+X1

    import code
    code.interact(local=locals())
    print("exiting interactive mode!!")
    
    pubkeys = [X1, X2, X3]
     
    #print("agg key:",xonly_point_agg(pubkeys))
    
    R1 = honest_signer.gen_partial_pubnonce()
    R2 = forged_signer.gen_partial_pubnonce()
    
    R3 = b"NEG" + R1 
    
    R = xonly_point_agg([R1, R2, R3])
    # print("agg nonce:",R)

    aggnonce = cbytes_from_point(R)
    sf = forged_signer.gen_partial_sig(pubkeys, aggnonce, msg)
      
    sig = bytes_from_point(R) + sf
   
    return pubkeys, sig
```

> Here $X_3$ is the negetive of `honest_signer` public key $X_1$.
On additionthe resultant will be $X_2$ which is `forged_signer` pub key.

Since in ECC there is no negetive as all values are in a fine field negetive points on ECC are represented simply by taking the complement of the $y$ cordinates i.e if $(x,y)$ is a point on the curve its negetive $-(x,y)$  is simply $(x,p-y)$ where $p$ refers to the finite fields size.

### MITIGATION

So to avoid this attack we would need to verify each persons public key with a message signed by the private key, But this would require each member of the multi-sig to do this thereby nullifying the scaling and efficiency of schnorr multisig scheme.

To solve this a new scheme was developed  called the **Bellare-Neven**
#### Bellare-Neven Scheme

in this scheme initially everyone shares their public key and a hash of the aggregated key is taken:

***signing***

$$\ell = Hash(P_{agg})$$

$$a_i = Hash(\ell||P_i)$$

$$P = \sum a_iP_i$$

$$R_{agg} = \sum R_i$$

$$e = Hash(R||P||m)$$

- Individual contribution to signature:

$s_i = z_i + k_ia_ie$

- Final aggregated signature is as usual:

$S = \sum s_i$

***verification***

$SG = R+eP$

- proof:

$$SG = \sum s_i G$$

$$= (z_i + k_ia_ie)G$$ 

$$= \sum z_iG + e\sum k_ia_iG$$

$$= \sum R_i + e\sum P_ia_i$$

$$= R + eP$$

Now trying the key cancellation attack on this scheme we can proove that this is secure against such attacks...

- Taking 2 party eg with `honest_signer` and `forged_signer`...

$P_{false} = P_f - P_h$

$R_{false} = R_f - R_h$

- This leads to both of them calculating the following “shared” values:

$$\ell = Hash(P_h||P_{false})$$

$$a_h = Hash(\ell||P_h)$$

$$a_{false} = Hash(\ell||P_{false})$$

$$P = a_{false}P_{false} + a_hP_h$$

$$R = R_h + R_{false} = R_f$$

$$e = Hash(R||P||m)$$

- forged sign is calculated as:

$s_f = z_f + k_se$

- To get the correct mulsig 

$s_fG = R + eP$

- Proving it wrong:

$$(z_f + k_se)G = R_f + e(a_{false}P_{false} + a_hP_h)$$

$$R_f + k_seG = R_f + e(a_{false}P_f - a_{false}P_h + a_hP_h) \tag{$R_f$ is common}$$

$$k_sGe = ea_{false}P_f - ea_{false}P_h + ea_hP_h \tag{$e$ is common}$$

$$k_sG = (a_{false}k_f - a_{false}k_h + a_hk_h)G \tag{$G$ is common}$$
<br>

$$k_s = a_{false}k_f - a_{false}k_h + a_hk_h$$

hence we can see we do not have al the information to get the value on the RHS hence the forged signature cannot be calculated and attack is mitigated.

## Wallet Key Reconstruction

### CHALLENGE

![](https://codimd.s3.shivering-isles.com/demo/uploads/b701362d73f35ebdd3fa8d3de.png)

Shares:
```
(1, 0xB4EB3F62388EA2343FFC28BB342D4245E9C8B3B0602825235460DD74F0D47AB1)
(3, 0xBE3A9EC4C3D5FA291CAFBB54C5F0301F29FE14539575408B57F3CF6C0EA6399E)
(4, 0x760E31377BF2240AD9DA8A9843FF4B27A0D42F66DF029BF06A8F584FA008EA4D)
```
The code used to generate the shares is as follows:


```python
import secrets

# Arbitrary upper bound
MAX_SHARES = 32

# Order of secp256k1 elliptic curve
p = 0xFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFEBAAEDCE6AF48A03BBFD25E8CD0364141
R.<x> = PolynomialRing(FiniteField(p))

def sss_get_shares(D, k, n):
    """
    Splits the secret value D into n shares; k of n are needed
    to reconstruct the secret.

    D is a 256-bit integer in [0, p)
    """
    assert 0 <= D < p
    assert 1 < k <= n <= MAX_SHARES

    q = D
    for i in range(1, k):
        a_i = secrets.randbelow(int(p))
        q += a_i * x^i

    return [(i, q(i)) for i in range(1, n + 1)]
```
                                  
### SOLUTION / IMPLEMENTATION

This was an easier challenge compared to the last one based on [shamirs secret sharing scheme](https://web.mit.edu/6.857/OldStuff/Fall03/ref/Shamir-HowToShareASecret.pdf)

Solve script:

```python
from libnum import *

p = 0xFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFEBAAEDCE6AF48A03BBFD25E8CD0364141

shares = [(1, 0xB4EB3F62388EA2343FFC28BB342D4245E9C8B3B0602825235460DD74F0D47AB1),(3, 0xBE3A9EC4C3D5FA291CAFBB54C5F0301F29FE14539575408B57F3CF6C0EA6399E), (4, 0x760E31377BF2240AD9DA8A9843FF4B27A0D42F66DF029BF06A8F584FA008EA4D) ]

res = 0
for i, pair in enumerate(shares):
    x, y = pair
    top = 1
    bottom = 1
    for j, pair in enumerate(shares):
        if j == i:
            continue
        xj, yj = pair
        top = (top * (-xj)) % p
        bottom = (bottom * (x - xj)) % p
    res += (y * top * invmod(bottom, p)) % p
    res %= p

flag = n2s(int(res))

print(flag)

#flag = correct! see you in the citadels

```
## CONCLUSION

I would really like to thank Elliot for making this amazing set of challenges while solving these I manged to learn a lot and all though there was some concepts I already knew I came across many new topics, some totally unrelated to the exploit for the challenges like the use of EC-schnorr in bitcoin, creating adresses in bitcoin, how to mitigate such attacks. Also came across a lot of papers and got to know about a lot more attacks/shortcomings of ECDSA/schnorr algorithms and how to correct them and their implementations in the bitcoin block chain. 

