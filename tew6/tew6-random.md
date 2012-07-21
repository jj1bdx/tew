# Erlang/OTP `random` module: issues and solutions<br>27-JUL-2012<br>Kenji Rikitake

---

## The `random` module

* Generating pseudo random number between [0, 1) （疑似乱数の生成）
* Simulating "unpredictable" events: testing the code, rolling a dice, generating noise （予測できない事象の生成に使う）
* It's **pseudo**: exact regeneration is possible （あくまで「疑似」なので再現が可能）
* Cryptographically *insecure* （**暗号学的には安全ではない**）

---

## For security: use `crypto` module

* Do not invent your own security module unless absolutely necessary （セキュリティ関連のモジュールを自力で作るのは極力避けるべし）
* Use `crypto:strong_rand_bytes/1` (OpenSSL wrapper) for secure pseudo random numbers, for Web Cookies, SSL/SSH private keys, etc.

---

## Issues on `random` module

The code is too simple and old (designed in 1982).

From R15B01 `random.erl`:

    !erlang
    -define(PRIME1, 30269).
    -define(PRIME2, 30307).
    -define(PRIME3, 30323).

    %%% internal state: 
    %%% only three 15-bit integers

    uniform() ->
        {A1, A2, A3} = case get(random_seed) of
                           undefined -> seed0();
                           Tuple -> Tuple
                       end,
        B1 = (A1*171) rem ?PRIME1,
        B2 = (A2*172) rem ?PRIME2,
        B3 = (A3*170) rem ?PRIME3,
        put(random_seed, {B1,B2,B3}),
        R = B1/?PRIME1 + B2/?PRIME2 + B3/?PRIME3,
        R - trunc(R).

---    

# I say again:<br>do not use `random` module for security application!

---

## A security hall of shame

* The following security holes are fixed by replacing `random` by `crypto`:
* CVE-2011-0766 - Erlang/OTP ssh module weak key generation leads into a case that an attacker can recover SSH session keys and DSA host keys. （SSHセッション鍵が予測可能）
* Yaws web server bugfix of 1.92 to 1.93 - an attacker can predict the next session ID for HTTP cookie-based sessions. （HTTPのCookieによるセッションIDが予測可能）

---

# Then why everybody does not use the `crypto` module functions?

---

## `crypto` module is expensive

* Internal entropy, the source of randomness gathered from unpredictable physical events, is a limited resource （エントロピーは有限）
* Cryptographically-safe random number generation is computationally expensive and slow （必要な計算量が多く遅い）
* Not all applications demand cryptographic security (e.g., simulation) （シミュレーションなど暗号学的安全性を必要としない野もある） 

---

## So why `random` is not enough?

* The generation period is too short (~2^43)
* The internal state size is too small (45 bits)
* The algorithm is suitable for simultaneously generating multiple independent/orthogonal streams, which means it cannot fully utilize the multi-core capability
* Other languages have already implemented modern PRNGs; why not on Erlang/OTP?

---

# We need some alternatives for `random` module

---

## Candidates

* SFMT: presented at ACM Erlang'11
* TinyMT: to be presented at ACM Erlang'12

---

## SFMT

* SIMD-oriented fast Mersenne Twister
* Generation period: 2^19937 - 1
* Internal state: 2496 bytes
* Not designed for generating multiple streams
* Chosen by PropEr as an alternative RNG
* My implementation available as `sfmt-erlang`
* Designed by Saito/Matsumoto, BSD licensed

---

## TinyMT

* Tiny Mersenne Twister
* Generation period: 2^127 - 1
* Internal state: 28 bytes
* Generating ~2^58 independent streams possible
* Seed jumping possible (for multiple streams)
* My implementation available as `tinymt-erlang`
* Designed by Saito/Matsumoto, BSD licensed

---

## Implementation lessons learned

* NIFnization is not necessarily fast; the function call and memory allocation overhead often becomes the bottleneck （NIF化で速くなるとは限らない）
* HiPE on x86_64/amd64 is fast (x2~3) for 32-bit bit manipulate operations （HiPEでビット演算が速くなることがある）
* The internal state size is mostly irrelevant to the performance （内部状態の大きさは速度にはほとんど関係がない）

---

## Goals and plans

* 

---


# Questions?
