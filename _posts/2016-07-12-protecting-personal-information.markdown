---
layout: post
title: "Protecting Personal Information"
date: 2016-07-12 08:00:00
categories: personal-information encryption golang
---

## What is PII?

Personally Identifiable Information (PII) has many competing definitions based
on who you ask, but in a nut shell it is information that can be used to 
identify a particular person, or a set of information that can be used in conjunction
to identify a particular person.  For example, a person's name combined with
a person's address would be considered personally identifiable information.

With the major data breaches in the past, where personal email addresses and 
names and addresses have been ex-filtrated from companies and even governments
it is truly our collective responsibilities as service providers to safe guard 
our customer's personal information from thieves that wish to take it from us.

It should be noted that my personal opinion is any company should only require
collection of the smallest amount of data required for the purpose of the service
provided.  This is obviously not popular, as it seems every internet service on
the planet needs ALL of the information about a user for no good reason, which
is why I do not partake in many popular online services.

## Goals

Given the importance of protecting personal information, our company had charged
us this past year to create an encrypted database of this sensitive information
with the following major goals:

1. Store all supporting data containing PII
2. Prevent a stolen database from being decrypted without authorization
3. Limit amount of data that can be ex-filtrated
  - Slow decryption
4. Allow easy extension and addition of new kinds of PII

## Implementation

With goals in hand we started designing a system in which can perform these goals:

### Slow Decryption

It is imperative that we need a mechanism for slowing the enemy down even if they
have the "master" key in hand.  AES256 takes a long time to decrypt if you do not
have the key, but if you have the key it is, by design, extremely fast decryption.

To accomplish this goal we focused efforts on the actual key derivation for the
slowdown knob.  By using a _unique password per account_ derived from a salted global
secret, we can have a unique password per account for our database records.

This by itself is nothing, as the password can be deterministic-ally generated 
with a small amount of knowledge of the process.  What we do now with this 
generated password is, we feed it through a key derivation function to generate
an AES256 key of 32 bytes.  Luckily for us there are two key generation 
functions that can provide the needed "slow down" capabilities we are seeking:
scrypt, and bcrypt.

Scrypt is a key derivation function that is CPU and Memory intensive, and bcrypt
is a key derivation function that is CPU intensive.  A typical cost of 10 on a
bcrypt operation can take hundreds of milliseconds just to generate the key.

This is exactly what we want for item number 3 on our list of goals.  We ended
up choosing to use scrypt for our implementation as it was very easy to provide
a unique salt to scrypt so that we could perform key derivation repeatably.
Bcrypt did not afford us this capability, and ends up requiring a "sample" record
to grab the salt from in order to perform a "compare".  Bcrypt is very much
more used in the industry as a password hashing mechanism, where you know already
what you want to compare it with.  As we want to keep the output of this hashing
function as the decryption key, we are not able to actually store the output of
bcrypt in order to get the salt that was used for the generation of the original
key easily without breaking up the hash from bcrypt.

In short, for this key derivation use-case scrypt made more sense for the time
being to generate the keys for the per record encryption.  Below is a snip-it of 
how we are using scrypt to generate a key for a record:

{% highlight go %}

// GenerateRandomSalt - create a new random salt
func GenerateRandomSalt() ([SaltSize]byte, error) {
	// read SaltSize random bytes into salt and return it
	salt := new([SaltSize]byte)
	err := PopulateRandom(salt[:], SaltSize)
	return *salt, err 
}

// GenerateKey - creates an encryption key using alg as the hashing algorithm
func GenerateKey(alg models.HashAlgorithm, key []byte) ([]byte, [SaltSize]byte, error) {
	// get salt = GenerateRandomSalt
	salt, err := genSalt()
	if err != nil {
		return nil, [SaltSize]byte{}, err 
	}
	
	// Hash(alg, salt, key+LatestGlobalSecret)
	hash, err := hashit(alg, salt, append(key, GlobalSecretKey...))
	if err != nil {
		return nil, [SaltSize]byte{}, err 
	}
	
	// return key and salt
	return hash, salt, nil 
}

// Hash - Given a HashAlgorithm, an optional salt, and plaintext reader, return a hash writer
func Hash(alg models.HashAlgorithm, salt [SaltSize]byte, plaintext []byte) ([]byte, error) {
	// perform hashing based on type, use salt if len(salt) > 0
	switch alg {
	case models.HashAlgorithm_SCRYPT:
		return Scrypt(salt[:], plaintext)
	case models.HashAlgorithm_SHA256:
		value := sha256.Sum256(plaintext)
		return value[:], nil
	case models.HashAlgorithm_SHA512:
		value := sha512.Sum512(plaintext)
		return value[:], nil
	default:
		return nil, errors.New("Failed to hash, not valid algorithm")
	}
}

// Scrypt - wrapper around scrypt
func Scrypt(salt, plaintext []byte) ([]byte, error) {
	return scrypt.Key(plaintext, salt, scryptN, scryptR, scryptP, KeySize)
}

{% endhighlight %}

With the above code we are able to call "GenerateKey" with a hashing algorithm 
and a generated password and we get back a byte slice of the encryption key, the
salt that was used to generate the hash, so we can have repeat-ability when trying
to recreate that encryption key, and an error.  The "Hash" function can accommodate
many types of hashing algorithms so if something else, maybe argon2, proves to be
better, it can easily be swapped into the application.

By using a salted global secret which is then scrypted, even an omnipotent 
hacker who has the global keys and knows which records belong to which accounts 
would have to take on the scrypt key derivation cost just to find the key in order 
to decrypt the records, which is non trivial with a huge database of users.

## Associations are bad for PII

As mentioned briefly at the end of the last section, associations are not good
in a PII data store.  In order to get around association leakage in our design
we decided that we would not store the actually account identifier, but rather
a one way hash representation of the account identifier and the _type_ of PII
element we were storing.

Much like the key derivation above, we are concatenating an per account salt
with a string representation of the PII type, i.e. "name", "address", and then
performing a sha512 hash of this merged value, which we store with the PII record.

This how we are minimizing the potential association among a user's PII 
information elements, as a user will have many different PII element types all
with a different lookup hash.

When we ask the database for a user's name records, we take that user's salt, 
append "name" to it, and run it through the "Hash" function above
to get back a hash value.  We then take that hash value and search the database
for exact matches, and those records are then decrypted by using the method 
explained in the previous section.

This gives us a good mechanism for non-associations in our data store, so an
enemy will not be able to perform targeted attacks on a particular account's
PII data.

## Encrypting and Storing

In order to store the data we first need to encrypt the data, and before that we
need to serialize the data in a structured way.  To accomplish this we turn 
to google's Protocol Buffers.  We take our data structure we defined with 
protocol buffers and marshal it to the wire format.

After marshaling we have to pad the plain-text.  We accomplish this with PKCS7
padding, and then feed it into the encryption mechanism with an algorithm:

{% highlight go %}
// padPKCS7 - pad input in using the PKCS7 algorithm and return results
func padPKCS7(in []byte) []byte {
	padding := 16 - (len(in) % 16)
	for i := 0; i < padding; i++ {
		in = append(in, byte(padding))
	}
	return in
}

// unpadPKCS7 - unpad in using PKCS7 algorithm and return results
func unpadPKCS7(in []byte) []byte {
	if len(in) == 0 {
		return nil
	}
	
	padding := in[len(in)-1]
	if int(padding) > len(in) || padding > aes.BlockSize {
		return nil
	} else if padding == 0 {
		return nil
	}
	
	for i := len(in) - 1; i > len(in)-int(padding)-1; i-- {
		if in[i] != padding {
			return nil
		}
	}
	return in[:len(in)-int(padding)]
}

	
// Encrypt - Given an EncryptionAlgorithm, a key, and plaintext, return a ciphertext
func Encrypt(alg models.EncryptionAlgorithm, key []byte, plaintext []byte) ([]byte, error) {
	// pad the plaintext to the correct block length with PKCS #7
	paddedPlaintext := padPKCS7(plaintext)
	
	// perform encryption and return ciphertext based on type
	switch alg {
	case models.EncryptionAlgorithm_AES256GCM:
		return AES256EncryptGCM(key, paddedPlaintext)
	case models.EncryptionAlgorithm_AES256CBC:
		return AES256EncryptCBC(key, paddedPlaintext)
	
	// return ciphertext or error
	default:
		return nil, fmt.Errorf("Encryption scheme not found: %s", alg)
	}

}

// Decrypt - Given an EncryptionAlgorithm, a key, and ciphertext, return a plaintext
func Decrypt(alg models.EncryptionAlgorithm, key []byte, ciphertext []byte) ([]byte, error) {
	// perform decryption and return plaintext based on type
	var plaintext []byte
	var err error
	
	switch alg {
	case models.EncryptionAlgorithm_AES256GCM:
		plaintext, err = AES256DecryptGCM(key, ciphertext)
		if err != nil {
			return nil, err
		}
	case models.EncryptionAlgorithm_AES256CBC:
		plaintext, err = AES256DecryptCBC(key, ciphertext)
		if err != nil {
			return nil, err
		}
	}
	
	// unpad the plaintext with PKCS #7
	unpaddedPlaintext := unpadPKCS7(plaintext)
	
	// return plaintext or error
	return unpaddedPlaintext, nil
}

// AES256EncryptGCM - gcm encryption mechanism
func AES256EncryptGCM(key, plaintext []byte) ([]byte, error) {
	// create new aes cipher
	block, err := aes.NewCipher(key)
	if err != nil {
		return nil, err
	}
	
	// create new gcm authenticator
	gcm, err := cipher.NewGCM(block)
	if err != nil {
		return nil, err
	}
	
	// generate a nonce
	nonce, err := GenerateNonce()
	if err != nil {
		return nil, err
	}
	
	// seal the plaintext with gcm.Seal, (make sure it appends nonce to beginning of ciphertext)
	ciphertext := gcm.Seal(nonce[:], nonce[:], plaintext, nil)
	
	return ciphertext, nil
}

// AES256DecryptGCM - gcm decryption mechanism
func AES256DecryptGCM(key, ciphertext []byte) ([]byte, error) {
	// create new aes cipher
	block, err := aes.NewCipher(key)
	if err != nil {
		return nil, err
	}
	
	// create new gcm authenticator
	gcm, err := cipher.NewGCM(block)
	if err != nil {
		return nil, err
	}
	
	// pull the nonce out of the ciphertext (first NonceSize bytes in ciphertext)
	// gcm open ciphertext with gcm.Open
	return gcm.Open(nil, ciphertext[0:NonceSize], ciphertext[NonceSize:], nil)
}

{% endhighlight %}

As can be seen above when we call "Encrypt" with an algorithm, a key, and plain-text
we get back the cipher-text.  When we call "Decrypt" with an algorithm, a key, 
and cipher-text we get back the plain-text representation.  

It is important to note that our goal of expand-ability is met here, as the plain-text
is just a serialized instantiated data structure, so the application logic should
know what it is and handle it appropriately.

## Conclusions

It truly is all of our responsibilities to maintain our customer's identifiable
information.  With a system described above you stand to have less enemy exposure
and more confidence in your data storage.  Even if you loose the global key, you
can have confidence that the data is still safe and secure.

Another point to remember is, please do not try to create a new encryption scheme!
In this system we use well known primitives for key generation and encryption.  
If you find yourself fighting with a standard library for encryption or hashing 
you are doing it wrong, please stop!

