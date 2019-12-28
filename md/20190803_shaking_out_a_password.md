# SHAKE'ing out a password

I think stateless password managers are a really cool piece of tech; often so simple to implement they can be conceived in just a few files of code, and yet have some _pretty ok_ security properties, relative to their complexity. The idea is simple: use a single master password, and at least one accompanying namespace string, and pass those contents though a KDF/hash func. The result is, given the same inputs, you can derive a deterministic password with strong security properties, one that is different per website/service name and different per login/username. All you need to remember is your master password and login details, and you can always determine your password, regardless of the machine you are on. 

Written as psuedo-code, that would look a little something like:
```
password = KDF(service, email, masterpassword)

KDF('company1', 'samuel.heslop.dev@gmail.com', 'masterPassword1') => 'abc...def123'

KDF('company1', 'samheslop@yopmail.com', 'masterPassword1') => 'jkl...mno456' // different email, different password

KDF('company2', 'samuel.heslop.dev@gmail.com', 'masterPassword1') => '...789' // different service, different password
```

In practice though, they aren't really fit as a replacement for a general purpose password manager; they don't handle breached passwords well (you'll have to remember some variance to apply to the KDF like a counter), there is no way to incorporate prior generated passwords (often meaning you need to keep your old stateful password manager around anyway), but perhaps most annoyingly, they simply fail when considering those silly requirements that sites impose on new passwords. You know, when you're told your password is too long, or that a 40 character password is insecure because it doesn't include at least one number. Oh, and you best hope none of your derived passwords get leaked in plaintext, else an attacker now has the means by which to begin attempting to crack your master password, and thus, every password derived from it.	

Regardless, as a little crypto exercise (and for a bit of fun), I decided to write my own implementation. My weapon of choice? Keccak!

## Keccak
Keccak is the core of the SHA-3 specification, chosen by NIST to represent the gold standard, and defacto choice for a modern hashing algorithm. It's sponge-like structure, and distinct separation between internal state and outputs automatically protects it against historical SHA weaknesses such as key extension attacks. The SHA-3 implementation constrains the outputs and rounds of the underlying system, meaning it loses a lot of the cooler elements of this sponge construct. While this is a good thing though, as it protects implementors from making bad decisions when using SHA-3 (like short outputs), one of the nicest elements of Keccak is the ability to 'absorb' a single input (in our case, a master password), and 'squeeze' out an arbitrary length output. Thankfully, in addition to the SHA-3 standard though, [NIST also released standards for derived functions](https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-185.pdf#27): SHAKE (SHA KEccak), cSHAKE (customisable SHAKE), TupleHash, ParallellHash and KMAC. The Go sha3 package exposes these primitives as well, and it is cSHAKE that I'll be basing my implementation on!    

```
cs := sha3.NewCShake256(nil, nil)
cs.Write(masterPassword)

var password []byte

for i := 0; i < 100; i++ {
  cs.Read(password) // each iteration, we receive a unique password
}
```

We could use this to derive multiple possible passwords from a single master password, like in the case we need our derived password to conform to a specified regex!

```
cs := sha3.NewCShake256(nil, nil)
cs.Write(masterPassword)

lastSqueeze := make([]byte, 16)
cs.Read(lastSqueeze)

for {
  // if password meets regex requirements, no more squeezes neccessary
  if r.Match(lastSqeeze) {
    return lastSqeeze
  }

  cs.Read(lastSqeeze) // squeeze another password out
}
```

This poses the question though, are we reducing our security here by performing multiple squeezes from only a single source of entropy? Fear not, for Keccak includes a permutation function that is performed once the length of read output has exceeded the size of the internal state. This permute allows indefinite expansion of the original material into a deterministic stream of bits. Also, I also notice some nice performance gains when squeezing more output rather than rehashing the input (though you'd generally expect the first operation to be more aggressively optimised for SHA-2):
```
BenchmarkSHA256Rehash-4                   10000000               224 ns/op
BenchmarkSHAKE256Squeeze-4                10000000               111 ns/op
```

## Utilising the Customisation String
You may have noticed the cSHAKE constructor func takes two parameters, []bytes name `N` and `S` respectively. The `N` is reserved for NIST defined functions (such as to differentiate between SHA3-128 and SHA3-256, ensuring different outputs). The `S` however, the 'customisation string', can be any value you like (provided it is less than 2<sup>2040</sup> in length). This is a great place for us to define our service names!

```
cs1 := sha3.NewCShake256(nil, []byte(`company1`))
cs2 := sha3.NewCShake256(nil, []byte(`company2`))

cs1.Write(masterPassword)
cs2.Write(masterPassword)

// The customisation string ensures these values will never match!
cs.Read(output1) 
cs.Read(output2)
```

SHAKE is stated to have identical security properties for operations using a customisation string and those not, as well as providing unrelating inputs across operations using a different customisation string, meaning there is no negative security implications for using this additional parameter.

## A few final touches...
Keccak is an uncosted primitive, meaning it isn't, in isolation, suitable for hashing passwords. To alleviate this, we can throw a costed KDF on our master password before 'absorption'. This is also a good time to incorporate our 3rd input, the 'login' (username, customer id, email address etc) for the account we are generating the password for. Argon2 is my go-to for a KDF. 

```
key := argon2.IDKey(masterPassword, login, 3, 128*1024, 4, 256)

cs := sha3.NewCShake256(nil, service)

cs.Write(key) // absorb!
```

Another design decision I made was to use base85 as the charset for my passwords; more possible characters means a massive increase to the number permutations an attacker would need to brute force, and the increase of symbols in the charset gives us an increased chance of generating a password that includes at least one symbol, something that often crops up in password requirement schemes. Let's just run some numbers on the comparison between 64 and 85 chars, assuming a 12 character password. Something to remember; increasing the number of available characters increases the base component, while increasing the length increases the exponent.

```
twelve := big.NewInt(12)
sixty4 := big.NewInt(64)
eighty5 := big.NewInt(85)

b64 := new(big.Int).Exp(sixty4, twelve ,nil)  //   4722366482869645213696 
b85 := new(big.Int).Exp(eighty5, twelve, nil) // 142241757136172119140625 (that's about 138 sextillion bigger)
```

## The end result!
Well that was fun! Here's my final version. There are a few things missing from this, like fixed-length support, but as I don't actually intend on ever using this (and you shouldn't either), it has served it's purpose of giving me some more experience playing with Keccak.

```
// arg 1 = service
// arg 2 = login (email address, username, customer id etc)
// arg 3 = password
// arg 4 = regex
func main() {
	result, err := execute(os.Args[1], os.Args[2], os.Args[3], os.Args[4])
	if err != nil {
		fmt.Printf("shakepass execution failed: %s", err.Error())
		return
	}

	fmt.Printf("%s", result)
}

func execute(company, login, masterPwd, regex string) ([]byte, error) {
	key := argon2.IDKey([]byte(masterPwd), []byte(login), 3, 128*1024, 4, 256)

	cs := sha3.NewCShake256(nil, []byte(company))
	cs.Write(key)

	out := make([]byte, 16)
	cs.Read(out)

	lastSqeeze := make([]byte, 20)
	ascii85.Encode(lastSqeeze, out)

	if regex != "" {
		r, err := regexp.Compile(regex)
		if err != nil {
			return nil, fmt.Errorf("invalid regex: %s", err.Error())
		}

		for {
			// if password meets regex requirements, no more squeezes neccessary
			if r.Match(lastSqeeze) {
				return lastSqeeze, nil
			}

			cs.Read(out)                    // squeeze another password out ...
			ascii85.Encode(lastSqeeze, out) // ...and base85 it
		}
	}

	return lastSqeeze, nil // no regex specifed, use first squeeze
}
```

Maybe I'll have some one-off use for something like this in the future, but I doubt it. Instead, I'll just consider this a fun exercise in using some fancy new primatives and happily continue using my real password manager :)