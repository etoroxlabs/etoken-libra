# A joyful early experiment of building eTokens on Libra 
*by Dr. Omri Ross, Peter Emil Jensen and Johannes Rude Jensen*

In this brief overview, we describe our initial experience with implementing the technology behind eToro tokenized assets, the ([eToken](https://github.com/etoroxlabs/etoken)) in the Move IR language for deployment on the Libra Network.

The Libra protocol is a deterministic state machine that stores data in a versioned database.  Using the novel domain specific language: Move. Move allows programmable transactions and modules that reuse code and state - similar to what we define as smart contracts.

Currently, Libra doesn't allow modules to be published, which is why the source code for eToken is meant
to be run as a test. The full source code for this Move IR eToken implementation can be found [here](https://github.com/etoroxlabs/etoken-libra/blob/master/src/eToken.mvir).

This test has been executed with the Libra version at commit hash `#004d472`.
The repository can be found [here](https://github.com/libra/libra/tree/004d472284ce7d3a0ce2f577ecd05de56026dbd4).

## The Move Intermediate Representation Language
With the release of Libra, a new domain-specific language named Move has been defined. 
Untill now, a higher level implementation has not been made available by the Libra team. With the recent announcement of the protocol, a less ergonomic intermediate representation of the lanaguge, dubbed 'Move IR', was released 

Move internalizes the idea of memory ownership and borrowing, very similar to how Rust operates. 
However, the novelty of the Move language is the way in which a 'resource' is defined. 

A 'resource' is a structure datatype that utilizes the ownership model, but can never be copied only moved and borrowed.
This is a core feature of the Move language as it guarantees that no defined resource accidentally duplicates, thus eliminating the possibility for double-spending or re-entrency attacks. 

Thus, it is sensible that the The notion of a 'resource' corresponds well with the concept digital assets

## eToken Implementation
The eToken is currently deployed on the Ethereum Blockchain and includes several important features implemented for the use of tokenization in production.

The most important features are listed below. The features in bold has been implemented in Move IR as well.

* **Roles (minters, blacklist)**
* **Minting**
* Burning
* Pausing
* Upgradability

We define a role as a capability in the Move implementation, the naming change is made to adhere to the standard of the Libra's own coin implementation.

### Capability
To be able to grant minter and blacklist permission, we must specify an owner of a module.
Being the owner of a module gives the user the ability to add accounts as minters and blacklists.

We start by defining the owner resource, which is only meant to be published once. 
It is declared in the `Capability` module. 
```
resource Owner { }
```
We then grant ownership by moving a published resource to the specified owner.
Here, we ran into some problems when trying to utilize the language to guarantee that the owner capability is only published once.

The Move IR definition does not seem to support functions defined as only executable once, during the initial module publish, also known as a constructor.

Naturally, this would have been an ideal place to grant 'owner' capability. 

Additionally, Move IR does not directly support global variables, which could be an unsafe way to define a function as already executed.

To circumvent these limitations, we created the module with a hardcoded owner address, creating a singleton resource.
Therefore an ownership grant is only performed if a capability resource is being published with the owner as a sender:

```
public publish() {
    let sender: address;
    sender = get_txn_sender();

    // Publish owner capability if sender is the privileged account
    // Replace 0x0 address with a real owner account address
    if (move(sender) == 0x0) {
        Self.grant_owner_capability();
    }

    ...
```

It is not possible for a module to publish resources on behalf of other accounts than the sender. Thereby, giving accounts full control of what is associated to them.
The function `publish()` therefore has to be executed by all accounts that wish to gain a valid capability, 
which is mandatory for further token usage. 

Neverthelss, enforcing a hardcoded owner address is not an elegant solution.
We addressed the Libra team with this concern, upon which a team member suggested implementing syntatic sugar replacing the hardcoded address, fx. `Self.publisher_address`.

The actual ownership granting is done by calling the internal function `grant_owner_capability()`, 
which creates an `Owner` resource and publishes it to the sender account. This is done by executing the following expression:
```
move_to_sender<Owner>(Owner {});
```
By implementing the said function as internal, the VM guarantees that it can only be executed by the module internally.
And by only calling the function when the sender is the specified owner address, we make sure that it is only published once.

The `publish()` also publishes a capability with no permissions for all accounts that call it, as required for further token usage.

It only reverts if it already exists.

By defining ownership as a resource, we can make sure that it cannot be copied. 
It also gives us a pleasurable type-safe way for us to secure privileged functions, such as granting others minting capability.
This is accomplished by simply requiring a borrowed `Owner` resource as a parameter to the privileged function.
An account can only acquire the borrowed reference by calling the `borrow_owner_capability()` function, 
which returns the borrowed reference if it exists at the sender address.
The following excerpt exemplifies an owner-privileged function:
```
// Grants minter capability to receiver, but can only succeed if sender owns the owner capability.
public grant_minter_capability(receiver: address, owner_capability: &R#Self.Owner) {
    let capability_ref: &mut R#Self.T;

    release(move(owner_capability));

    ...
```
The borrowed owner capability is only used for type-security and is therefore immediately released to the sender:
If the function successfully executes, it mutates the capability resource located at the `receiver` address.
```
    ...

    // Pull a mutable reference to the receiver's capability, and change its permission.
    capability_ref = borrow_global<T>(move(receiver));
    *(&mut move(capability_ref).minter) = true;

    ...
```

### Token
With a capability module in place defining the roles and permissions for the eToken, 
we can now proceed with the actual token implementation.

We start by declaring the token resource, which holds the number of said tokens.
```
resource T {
    value: u64,
}
```

This is where the benefits of Move, in comparison to other smart-contract languages, becomes apparent.
If we were to deposit a number of tokens, we have to control the memory ownership of the tokens.
We can only gain this ownership by splitting an existing owned token (also known as withdrawing) or when minting fresh tokens.

This ownership property guarantees that the same tokens cannot exist elsewhere, thus eliminating bugs stemming from incorrect duplications allowing double-spending and other erroneous behavior.

Furthermore, the memory ownership model also requires that an owned token has to be either explicitly destroyed or moved to another owner. This guarantees that the token doesn't get accidentally locked inside a module and never to be retrieved again.

By utilizing this type-safe property we can define the function for depositing owned tokens.
```
public deposit(payee: address, to_deposit: R#Self.T, capability: &R#Capability.T) {
    
    ...

    Capability.require_not_blacklisted(move(capability));

    payee_token_ref = borrow_global<T>(move(payee));
    payee_token_value = *(&copy(payee_token_ref).value);

    // Unpack and destory to_deposit tokens
    T{ value: to_deposit_value } = move(to_deposit);

    // Increase the payees balance with the destroyed token amount
    *(&mut move(payee_token_ref).value) = move(payee_token_value) + move(to_deposit_value);

    ...
```

We start by making sure that the user is not blacklisted. 
Next, we destroy the owned tokens by unpacking its inner quantity variable. 
Last we increase the payee's tokens by the unpacked amount.

In contrast to depositing, when withdrawing tokens from the sender's account
we split the tokens into two pieces and return ownership of the new tokens to the caller.

The splitting is done by first decreasing the senders token amount, and then returning a new token resource.
```
*(&mut move(sender_token_ref).value) = move(value) - copy(amount);
return T{ value: move(amount) };
```

## Conclusion
All in all, Libra and Move IR is a welcome step forward in smart-contract development.
Having strong asset-guarantees helps developers to produce less error-prone code and move faster.

Nonetheless, Move IR is still in an early stage and is not user-friendly in its current iteration. It is called an 'intermediate representation' for a reason :-)

We will follow the development closely and look forward with excitement to new developments.

## Try and Run the Test
If you are interested in learning more abou the Libra network and Move IR, we recommend running the test to familiarize yourself with the concepts described above. To run the crude eToken implementation test, you should perform the following steps:

* Clone the Libra repository (preferably at the commit hash stated in the introduction)
* Follow the Libra `README` for how to compile and install it.
* Clone this repository
* Copy the eToken Move IR source code located at `src/eToken.mvir` in this repository, 
  to the test folder in the Libra repository located at `language/functional_tests/tests/testsuite/module/`
* Execute the following command somewhere in the Libra repository: `cargo test -p functional_tests eToken`

## Comment or Reach Out

Thanks for reading. If you are interested in learning more about eToro and eToroX alongside our work with digital asset infrastructure, we recommend keeping an eye out in our [GitHub](https://github.com/etoroxlabs), where we rountinely open-source our work.

We are aways looking for talented developers with a passion for distributed infrastructure. If you are a senior developer with a passion for functional programming, formal verification techniques or any of the many exciting issues related to these disciplines, feel free to reach out here or write Omri@etorox.com or Johannesje@etorox.com.

Thanks and cheers from the eToroX Labs team.
