+++
date = '2025-11-30T20:27:07+05:30'
draft = false
title = 'MPC: The Method Custodial Wallets Should Use for Key Management'
+++

Let cut part where I explain what is MPC? because So there are lot of resources covering that theory part.
here we are gonna do implemenation of mpc based key signing with one of the open
source repository from coinbase [cb-mpc](https://github.com/coinbase/cb-mpc). well this repostiory is written in C++.
Well fear not we are not gonna do this in C++ (because I don't like C++ enought to write code in it).
one thing i want to share is it took lot of time to find this perfect library. because if you go and serach for a mpc-library in go or any other language
there is one feature is missing. that is hd-key deriviation. I work on bitcoin based blockchain so I needed a mpc which does hd key sign. well the library that we are
using is not mpc in terms of hd key because it have this 2pc approach means 2:2 mpc, which is not real threshold based mpc but its better than others(others which i didn't find. lol!!)

Let's start with single key MPC and hd key 2pc, note that I said we are not gonna do this with C++ where the library is in C++.
how then? we are gonna call the C++ with C equivalent wrapper and call that C with go. why this C++->C->Go hassle you may ask that is because C++ [ABI]() is sh\*t.

# Pre-Requisite

If you are C++ and Go expert skip this section because I dont want to embarass myself presenting you how to install the library on your system.
That is you need clone the [cb-mpc](https://github.com/coinbase/cb-mpc). use clangd 20 as recommended rather than gcc.
library and do `make openssl-linux`(for openssl ) ,`make build-no-test` and `make install` (for linux), this will install the library in opt/ folder so
its not in gcc/clang path so you need to include it during compilatiation. one easy way is I use a [direnv]() so i kept a .envrc file on root of the github repo with values

```bash
export CC=clang
export CXX=clang++

export CGO_CFLAGS="-I/usr/local/opt/openssl@3.2.0/include -I/usr/local/opt/cbmpc/include"
export CGO_LDFLAGS="-L/usr/local/opt/openssl@3.2.0/lib64 -lcrypto -lssl -L/usr/local/opt/cbmpc/lib -lcbmpc -lpthread -ldl"

export CPATH="/usr/local/opt/openssl@3.2.0/include:/usr/local/opt/cbmpc/include:$CPATH"
export LIBRARY_PATH="/usr/local/opt/openssl@3.2.0/lib64:/usr/local/opt/cbmpc/lib:$LIBRARY_PATH"
export LD_LIBRARY_PATH="/usr/local/opt/openssl@3.2.0/lib64:/usr/local/opt/cbmpc/lib:$LD_LIBRARY_PATH"
```

Now after you done the setup, you can see there is a directory named demos-go. open it and go there you find cb-mpc-go.
This is the thing you want to use for go to call the library apis. I did encounter error on this internal/cgobinding/cmem.h. where you can see that
path of the cb-mpc library path is hardcoded so it will give error later. just remove the hardcoded path or just put your path there. rest should work

# MPC: single key pair signing

Now you are ready go with the cb-mpc-go directory. import this in your project. if you see that there is demos-go/cmd/thereshold-signing example in the repo
which does the single key pair singing better way than i could ever explain. go and look into that follow the makefile and you are given a web interface
for generaing keypair and singing a message.

Now I will explain this code in my own words, what they are doing and how is it working. this will be a long blog(only my neovim keeping me writing this. lol)
Let explain this in a diagram which i will generate from llm.

```


```

this post is in draft-release phase.
