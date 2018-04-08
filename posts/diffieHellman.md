# Diffie Hellman Key Exchange in Rust

Today we are going to go over a simple implementation of the Diffie Hellman Key Exchange algorithm.

The algorithm can be used to provide "forward security" in TLS. The newer Elliptic-Curve Diffie-Hellman Key exchange algorithm should be used in practical applications though. 

The now-expired patent from 1977 is here, its a pretty good read:

https://patents.google.com/patent/US4200770

## Initializing The Binary Project with Cargo

We initialize our Rust binary application with Cargo, Rust's package manager.

```cargo init --bin``` 

This will create the following file src/main.rs:

```
fn main() {
	println!("Hello, world!");
}
```

## The Boilerplate

To accept command line arguments, such as the users secrets they have chosen (we will be using integers in this example), we need to use `std::env`. At the top of src/main.rs, do:

```use std::env;```

Next, make our main method look something like this:

```
fn main() {
	let args: Vec<String> = env::args().collect();

	let alice_secret = &args[1];
	let bob_secret = &args[2];

	println!(
		"Alice's Secret: {}, Bob's Secret: {}",
		alice_secret,
		bob_secret
	);
}
```

Now if we run the project with command line arguments:

```cargo run 2 3```

We should get output that looks like this:

```Alice's Secret: 2, Bob's Secret: 3```

## The Diffie-Hellman Key Exchange Algorithm

This algorithm is as simplistic as it is powerful. It allows the generation of secret keys over an insecure channel, without ever transmitting those keys over the wire.

Here is a description of how the algorithm works in this example:

* Alice and Bob agree to use a given prime number, p, in this case we will use 23. They also agree upon a base g=5.

* Alice chooses a secret integer a = 6, then sends Bob A = ga mod p
* This becomes: A = 56 mod 23 = 8

* Bob chooses a secret integer b = 15, then sends Alice B = gb mod p

* This becomes: B = 515 mod 23 = 19

* Alice computes s = Ba mod p

* Which is: s = 196 mod 23 = 2

* Bob computes s = Ab mod p

* Which is: s = 815 mod 23 = 2

* Alice and Bob now share a secret (the number 2)

Whoa that was a lot of math! You may be wondering why an outside observer wouldn't be able to break this, since we have transmitted most of the parameters for generating the shared secrets. The simplest explaination is that we have done all of our math under some modulus p, and for an outside observer to break our encryption they would need to solve whats known as the discrete logarithm problem. 

A good explaination of the discrete logarithm problem is here:

https://wstein.org/edu/124/lectures/lecture8/html/node5.html

Essentially, the larger prime we use for p, the harder our encryption is to crack!

## Implementing The Algorithm

We need to create a function that takes the power of some value, under some modulus, below is the function we will use:

```
fn power_under_modulus(a: u64, b: u64, modulus: u64) -> u64 {
	let temp: u64;

	/* If b is 1 then we just return a */
	if b == 1 { 
		return a;
	}
	/* We reduce the value down further under our modulus */
	temp = power_under_modulus(a, b/2, modulus);

	/* If b is even, return temp^2 % modulus */
	if (b % 2) == 0 { 
		return (temp * temp) % modulus;
	} else {
		/* If its odd, return this */
		return (((temp * temp) % modulus) * a) % modulus;
	}
}
```

Now in src/main.rs we implement the algorithm after we print Alice and Bob's secrets out. Our main method should look like this now:

```
fn main() {

	let args: Vec<String> = env::args().collect();

	let alice_secret = &args[1];
	let bob_secret = &args[2];

	println!(
		"Alice's Secret: {}, Bob's Secret: {}",
		alice_secret,
		bob_secret
	);

	/* We want to use the same n and g for Alice and Bob */
	/* Our prime number */
	let p = 23; 
	/* Our agreed upon base */
	let g = 5; 

	/* Parse unsigned integer from the secrets */
	let alice: u64 = alice_secret.parse::<u64>().unwrap();
	let bob: u64 = bob_secret.parse::<u64>().unwrap();
	
	/* Calculate the values to be passed
	 * through the insecure channel 
	 */
	let a = power_under_modulus(g, alice, p);
	let b = power_under_modulus(g, bob, p);

	let alice_shared_secret: u64 = power_under_modulus(b, alice, p);
	let bob_shared_secret: u64 = power_under_modulus(a, bob, p);

	println!(
		"The shared secret for Alice is: {}", 
		alice_shared_secret
	);
	println!(
		"The shared secret for Bob is: {}",
		bob_shared_secret
	);
}
```

Now if we run it, we should get output like this:

```cargo run 20 30```

```Alice's Secret: 20, Bob's Secret: 30```

```The shared secret for Alice is: 8```

```The shared secret for Bob is: 8```

These are pretty small values, and pretty easy to crack. Try setting p to some large prime number (within 64 bits) and try larger integers for alice and bob and see how the shared secret size grows, increasing security. 

Thanks for reading!


