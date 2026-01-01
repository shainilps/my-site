+++
date = '2025-11-30T20:27:07+05:30'
draft = false
title = 'So You Want MPC in Go: Threshold Signing and HD Keys with cb-mpc'
+++

NOTE: This is not MPC from scratch or algorithmic level discussion. here I'm explaining a mpc library more specifically [cb-mpc](https://github.com/coinbase/cb-mpc) from coinbase.

Let's cut part where I explain what is MPC? because there are lot of resources covering that theory part.
here we are gonna do implemenation of mpc based key signing with one of the open source repository from coinbase [cb-mpc](https://github.com/coinbase/cb-mpc).
well, this repostiory is written in C++. Well fear not we are not gonna do this in C++ because I already did this with Go, so we are gonna use it.
one thing I want to share is it took lot of time to find this perfect library for my usecase because if you go and serach for a mpc-library in go or any other language
there is one feature is missing, that is HD-key deriviation. I work with bitcoin based blockchain so I need a mpc lib which does HD key sign. well, the library that we are
using is not MPC in terms of HD key because it have this 2PC approach means 2:2 mpc, which is not real threshold based MPC but its better than others(others which i didn't find. lol!!)

Let's start with single key MPC and HD key 2PC. Note that I said we are not gonna do this with C++, even though the library is written in C++.
How? we are gonna call the C++ with C equivalent wrapper and call that C with go. why this C++->C->Go hassle you may ask that is because C++ [ABI]() is sh\*t.

# Pre-Requisite

If you are C++ and Go expert you may skip this section and read the `README.md` of the library which are supposed to install and follow the instruction to set it up.
This is the repo: [cb-mpc](https://github.com/coinbase/cb-mpc). use clangd 20 as recommended rather than gcc.
library and do `make openssl-linux`(for openssl ) ,`make build-no-test` and `make install` (for linux), this will install the library in opt/ folder so
its not in gcc/clang path so you need to include it during compilatiation. one easy way is I use a [direnv](https://direnv.net) so I kept a `.envrc` file on root of the github repo with values

```bash
export CC=clang
export CXX=clang++

# for  go compilation of CGO
export CGO_CFLAGS="-I/usr/local/opt/openssl@3.2.0/include -I/usr/local/opt/cbmpc/include"
export CGO_LDFLAGS="-L/usr/local/opt/openssl@3.2.0/lib64 -lcrypto -lssl -L/usr/local/opt/cbmpc/lib -lcbmpc -lpthread -ldl"

#  for the library to available in system path
export CPATH="/usr/local/opt/openssl@3.2.0/include:/usr/local/opt/cbmpc/include:$CPATH"
export LIBRARY_PATH="/usr/local/opt/openssl@3.2.0/lib64:/usr/local/opt/cbmpc/lib:$LIBRARY_PATH"
export LD_LIBRARY_PATH="/usr/local/opt/openssl@3.2.0/lib64:/usr/local/opt/cbmpc/lib:$LD_LIBRARY_PATH"
```

Now after you done the setup, you can see there is a directory named `demos-go`. open it and go there you find `cb-mpc-go`.
This is the thing you want to use for go to call the library apis. I did encounter error on this `internal/cgobinding/cmem.h`. where you can see that
path of the cb-mpc library path is hardcoded so it will give error later. just remove the hardcoded path. rest should work. If you still need some help please do consider
contacting me through twitter or email.

# MPC: Single Key Pair

Now you are ready to go with the `cb-mpc-go` directory. import this in your project. if you see that there is `demos-go/cmd/thereshold-signing` example in the repo
which does the single key pair singing better way than I could ever explain. go and look into that follow the makefile and you are given a web interface
for generaing keypair and singing a message. In this whole section, I will explain how the code is working with some snippets when needed.

## First Step: DKG(Distributed Key Generation)

As it says this is how we generate keyshare. Each party have it's own keyshare. This keyshare is each party of is
used during signing. Important note is in the example they have not shared any better way to store keyshare because keyshare is just an object and there is no standard
serialising format(This problem also exist in go-tss library, which make use of json.marshal() to serailise struct). The library itself claims that this is not safe but
they haven't provided any alternatives. You will see `keyshare.Marshal()` to convert that into raw bytes.

How does the communication work between parties ? What is the medium ? This is very interesting, the transport channel for the library is not restrictive. They provide a interface
which we can implement anyway we want. In the `apis` there exist a mock and TCP+MTLS(MTLS means [Mutual TLS](https://www.cloudflare.com/learning/access-management/what-is-mutual-tls)) implementations. The provided Mutual TLS implementation
can only do one session at a time. if you want something more wrap session the MTLS. The Interface of Transport is named `Messenger` the below code snippet shows how it looks.
This is basically how each party communicate with each other. Each party maintains a single connection for each other in any channel then read and write to it.

```go
// Messenger defines the interface for data transport in the CB-MPC system.
// Implementations of this interface handle message passing between MPC parties.
type Messenger interface {
	// MessageSend sends a message buffer to the specified receiver party.
	MessageSend(ctx context.Context, receiver int, buffer []byte) error

	// MessageReceive receives a message from the specified sender party.
	MessageReceive(ctx context.Context, sender int) ([]byte, error)

	// MessagesReceive receives messages from multiple sender parties. It waits
	// until all messages are ready and returns them in the same order as the
	// provided senders slice.
	MessagesReceive(ctx context.Context, senders []int) ([][]byte, error)
}
```

There is another thing called **Job**. each operation have it's own job. this job takes the messenger interface to communicate with each other. Why Job exist? Inorder
maintain a internal session so that concurrent execution won't interfere with each other in library level(application level you need to seperate it). each job have it's
own session id(some time it's generated internally or else it takes during operation such as dkg or sign).

This job is passed to the DKG function with one more important parameter **Access Structure**. Access Structure is the way in which we can specify
how the way key share is split and will be will be used. This specifies how many parties are there ? What is the threshold ? Not only this It also takes the
party names, party kind (which is `KindThreshold` in our case). why party kind this let's you specify more about threshold for example we can say that party 2 is needed when
5:3 threshold based signing. `KindThreshold` doesn't enforce this. It says, Any m of n party can sign it.

Now let me show you a snippet of code which does the DKG

```go
    //quoramCount is party count
    //partyIndex is the current party index
    //quorumPNames is names of the parties
    //messenger is  the transport interface
	job, err := mpc.NewJobMP(messenger, totalPartyCount, partyIndex, allPNameList)
	if err != nil {
		return nil, fmt.Errorf("failed to create job: %v", err)
	}
	defer job.Free()

	if quorumCount < 2 {
		return nil, fmt.Errorf("threshold must be at least 1")
	}
	if quorumCount > len(allPNameList) {
		return nil, fmt.Errorf("threshold must be less than or equal to the number of online parties")
	}

	sid := make([]byte, 0) // empty sid means that the dkg will generate it internally
	keyShare, err := mpc.ECDSAMPCThresholdDKG(job, &mpc.ECDSAMPCThresholdDKGRequest{
		Curve: curve, SessionID: sid, AccessStructure: ac,
	})
	if err != nil {
		return nil, fmt.Errorf("threshold DKG failed: %v", err)
	}

```

## Second Step: Threshold based Signing

Signing is the operation in which we sign a message with the m:n threshold manner. since, I already explained the Messenger, Job and Access Structure. Let's explain what different
comes here. Only thing that is different would be the additive share that we add to the keyshare before signing. This is inorder to satisfy the access strutcure which was provided
during DKG.
Note that during signing, we need to specify which party gets the signature response; not all party gets the signature response-only one does.

```go

    //quoramCount is party count
    //partyIndex is the current party index
    //quorumPNames is names of the parties
    //messenger is  the transport interface
	job, err := mpc.NewJobMP(messenger, quorumCount, partyIndex, quorumPNames)
	if err != nil {
		return nil, nil, fmt.Errorf("failed to create job: %v", err)
	}
	defer job.Free()

	if quorumCount != len(quorumPNames) {
		return nil, nil, fmt.Errorf("quorum count does not match the number of participants")
	}

    // the Message should be sha256 hash(this is optional here we can do this outside of the mpc server)
	hashedMessage := sha256.Sum256(inputMessage)
	message := hashedMessage[:]

	//key share is the object we got from key generation response
    //ac is basicaly AccessStructure, that we generate during DKG
    //quorumPNames participant parties names
	additiveShare, err := keyShare.ToAdditiveShare(ac, quorumPNames)
	if err != nil {
		return nil, nil, fmt.Errorf("converting to additive share: %v", err)
	}

    //LEADER_INDEX is basically tells the mpc operation in which party signature should be returned (kinda like master node)
	sigResponse, err := mpc.ECDSAMPCSign(job, &mpc.ECDSAMPCSignRequest{
		KeyShare:          additiveShare,
		Message:           message,
		SignatureReceiver: LEADER_INDEX,
	})
	if err != nil {
		return nil, nil, fmt.Errorf("signing failed: %v", err)
	}

```

# 2PC: HD Key Pair

If you are already read the first half you are already familiar with the most of the terminology needed for this section. considering you know what is communication layer and job, we will move forward with our 2PC approach of hd key pair.

Now what is 2PC ? well in regular threshold based signing you set m:n (example 2:3 read as 2 out of 3). but in 2pc its always 2:2. yes there is no threshold and there is no `n` also. I won't say it's totally bad because MPC isn't developed enough to make this happen. child key derivation is hard to do in MPC. so the creators of this library develeped a way we can achieve hd key pairs. Another thing to note is this makes use of custom BIP-32ish derivation scheme. Okay you heard me right it's bip32ish not bip32 either. because bip-32 can't be achieved due to the limitation of mpc. But This is also look exactly like bip32 and safe to use. you can read about how this works [here](). Now let us move on to the actual code.

## First Step: DKG(Distributed Key Generation for 2PC)

This is very similar to MPC as we did before. This also creates a **2PC job** to interact with parties (2 parties). One difference to note is that here we are not getting any actual key as response on DKG. the key object we get is not an actual key share like we got previously with DKG. This is the root key share which is used for deriving other keys in the bip32 manner.

```go
    //partyIndex is the current party index
    //pnames is names of the parties
    //messenger is  the transport interface

	job, err := mpc.NewJob2P(messenger, partyIndex, pnames)
	if err != nil {
		return fmt.Errorf("failed to create job: %v", err)
	}
	defer job.Free()

	//curve is secp256k1 curve object.
	keygenResponse, err := mpc.HD2PCKeyKeyGen(job, &mpc.HD2PCKeyKeyGenRequest{
		Curve: curve,
	})
	if err != nil {
		return nil, fmt.Errorf("threshold 2pc DKG failed: %v", err)
	}


```

## Second Step: Deriving Child Keys

Deriving Keys is the new thing but as you guessed this also works in the same manner. you create a **2PC job** and let them do the job. one thing to note is there is no need of you to derive each individual operation, we can derive multiple keys at once . Let's look onto the code you will get beter understanding.

```go
    // partyIndex is the current party index
    // pnames is names of the parties
    // messenger is  the transport interface
	job, err := mpc.NewJob2P(messenger, partyIndex, pnames)
	if err != nil {
		return fmt.Errorf("failed to create job: %v", err)
	}
	defer job.Free()

	// hdRootKey is the object we got from dkg response
	// sessionID is a unique random bytes
	// hardened path is basically an array of []uint32
	// nonhardened path is [][]uint32 (so this is the reason i said we can derive mulitple keys at once)
	// deriveReponse will have an array of key objects instead of single key object
	deriveResponse, err := mpc.HD2PCKeyDerive(
		job,
		&mpc.HD2PCKeyDeriveRequest{
			SessionID:        []byte("random-bytes"),
			KeyShare:         hdRootKey,
			HardenedPath:     mpc.BIP32Path{0x8000002C, 0x80000000, 0x80000000}, //44'/0'/0'
			NonHardenedPaths: []mpc.BIP32Path{{0, 1}, {0, 2}},
		},
	)
	if err != nil {
		return nil, fmt.Errorf("key derivation failed: %v", err)
	}

```

## Final Step: Signing in 2PC approach:

If you really read this far then you already know how is this signing gonna work, because signing also works like pretty much it did before with MPC. one change would be there is not **Access Structure** due to both party needs to sign.

```go
    // partyIndex is the current party index
    // pnames is names of the parties
    // messenger is  the transport interface
	job, err := mpc.NewJob2P(messenger, partyIndex, pnames)
	if err != nil {
		return fmt.Errorf("failed to create job: %v", err)
	}
	defer job.Free()


	// session id is a random id (should be unique)
	// KeyShare is the child Key object we got from the key derviation step
	//  message bytes is the data that you are supposed to sign
	signedRes, err := mpc.ECDSA2PCSign(
		job,
		&mpc.ECDSA2PCSignRequest{
			SessionID: []byte("random-bytes"),
			KeyShare:  hdChildKey,
			Message:   []byte("message-bytes to sign"),
		},
	)
	if err != nil {
		return nil, fmt.Errorf("signing failed: %v", err)
	}

```

That's it. you can now work with this MPC library very easily. One more thing I want to say is if the go bindings arent in present in the demos-go for some functions then build your own bindings and enjoy.

Since you got this far and I'm writing this in 1 Jan 2026, I'm feeling very generous. here is some more code which you can probably get from a LLM but still giving it anyway.

```go
//pem to ec key
func pemToECKey(pemStr *string) (*ecdsa.PublicKey, error) {
	if pemStr == nil {
		return nil, errors.New("pem is nil")
	}

	block, _ := pem.Decode([]byte(*pemStr))
	if block == nil {
		return nil, errors.New("failed to decode ")
	}

	var pki publicKeyInfo
	_, err := asn1.Unmarshal(block.Bytes, &pki)
	if err != nil {
		return nil, err
	}

	asn1Data := pki.PublicKey.RightAlign()
	_, x, y := asn1Data[0], asn1Data[1:33], asn1Data[33:]
	x_big := new(big.Int)
	x_big.SetBytes(x)
	y_big := new(big.Int)
	y_big.SetBytes(y)

	ecPubKey := ecdsa.PublicKey{
		Curve: crypto.S256(),
		X:     x_big,
		Y:     y_big,
	}

	return &ecPubKey, nil
}
```

bye, happy hacking <3
