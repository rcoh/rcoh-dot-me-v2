---
title: "Demystifying Two Factor Auth"
date: 2018-01-06T23:36:23-08:00
draft: false
tags: ["no-magic", "python", "security"]
---
I always wondered how Google Authenticator style 2-factor codes worked. The process of going from QR code to rotating 6-digit pin seemed
a bit magical. A few days ago, my curiosity found itself coupled with some free time. Here's what I found:

### What's in the QR Code
I scanned the QR code from Github with a barcode scanning app. Here's what's inside:

```
otpauth://totp/Github:rcoh?secret=onswg4tforrw6zdf&issuer=Github
```

Not too surprising. It tells us the protocol, [TOTP](https://tools.ietf.org/html/rfc6238), who is issuing this OTP code (Github), and most importantly the secret:[^3]

```python
secret = 'onswg4tforrw6zdf'
```

So how do we go from this to a time rotating 6-digit code?

### From Secret to 6-digit Code
I followed the spec and was able to implement a [Python script](https://gist.github.com/rcoh/c4dc45825a322881a9c1b300d70c3941) that generated the same codes as Google Authenticator. Before we dive in, a quick overview:

1. Figure out your `time_chunk` based on the current time.
2. Generate an hmac that combines your `time_chunk` and your secret. Combining these with HMAC is a convenient way to prove that you have the     secret without revealing it. It's a sha1-hmac, which means we'll get a 20-byte digest.
3. Converting the digest into a number has two steps:
  1. Take the last 4-bits of the digest. These become the offset (back into the digest). Eg. if the last 4 bits are `0010`, then we would start at the 4th byte.
  2. Read 4 bytes starting from the offset.
4. Convert the 4 byte section into a *n* digit integer. This is your two-factor code.

Here's how that works in practice:

1. The algorithm works by finding what 30 second time slice we're in, starting from the Unix Epoch:

    ```python
    time_chunk = int(time.time() / 30)

    # To feed this into the next step, we need the binary representation.
    # Per the spec, the time is represented as an 8-byte string in
    # big endian. '>q' gets us a  big-endian unsigned long (8-bytes)
    time_bytes = struct.pack('>q', time_chunk)
    ```
    Most server implementations will accept any time chunk within a fairly small range of surrounding 30 second buckets to allow for clock skew.

2. Next, feed our time_chunk in the [HOTP](https://tools.ietf.org/html/rfc4226) algorithm. HOTP is an algorithm for using HMAC to generate one-time-passwords based on a counter. In the case of TOTP, the counter is the time chunk we're in. First, we need to create an hmac of our secret key and the count:

    ```python    
    key = base64.b32decode(secret.upper()) # Python only b32decodes upper case
    hm = hmac.new(key, time_bytes, hashlib.sha1)
    hex_digest = hm.hexdigest()
    ```

    *Side note: The default hash function in Python for HMACs is MD5!*[^2]

3. Armed with the hex digest, we next need to determine the offset in the array we'll be reading from. This offset is chosen based on the last 4-bit chunk of the array:

    ```python
    # Take the last 4 bits. These become the offset into the array
    offset = int(hex_digest[-1:], 16)
  ```

4. Starting at the offset, read 4 bytes. This will be the basis of our 6 digit code:

    ```python
    # Take 4 bytes starting from the offset (in bytes). Our array of hex digits is 1/2 byte
    # per character, hence offset*2 and +8
    relevant_bytes = int(hex_digest[offset*2:offset*2+8], 16)
    ```

5. Drop the highest order bit (the sign). Dropping this makes it easier to implement in different programming languages which may have differing behavior on signed module operations.[^1]

    ```python
    unsigned_bytes = relevant_bytes & 0x7FFFFFFF
    ```

6. Last but not least, take the unsigned `result % 10**d` where d is the number of digits you want. In our case, that's 6:

    ```python
    num_digits = 6
    final = masked % (10 ** num_digits)
    # left pad zeros for ease of use
    final_str = str(final).zfill(6)
    return final_str
    ```

### Backup Codes
It's not in the spec as far as I saw, but Google Authenticator style codes usually also include backup codes. The [Google implementation](https://github.com/google/google-authenticator-libpam/blob/master/src/google-authenticator.c) derives them by taking 4-byte sections of the secret and converting them into digits by the same process as the time-based system: `bignum % (10 ** d)`. This means they're also derivable in a predictable way from the original key so they don't need to by stored separately by the server. Some other systems use hex-based codes instead, but presumably it's a similar process.

### Parting Thoughts

Should you implement 2-factor auth, make sure your comparison method is safe from timing attacks. I did a quick survey of 2-factor libraries and of libraries that offered `verify` methods, most got it right. The easiest way to compare 2-factor codes in a constant time way is convert to `int`s first. [Google does it](https://github.com/google/google-authenticator-libpam/blob/master/src/pam_google_authenticator.c#L1385), so I think it's safe to say that's legit.

If you want to find out when your 2-factor code will be your favorite number, you can use [this handy Python script](https://gist.github.com/rcoh/c4dc45825a322881a9c1b300d70c3941).

***
{{% subscribe %}}

[^1]: Python and Ruby differ from C and Golang, for instance: https://github.com/golang/go/issues/448

[^2]: Having Md5 as the default for hmac isn't a complete disaster, but it isn't great. According to [RFC 6151](https://tools.ietf.org/html/rfc6151) "attacks on HMAC-MD5 do not seem to indicate a practical vulnerability when used as a message authentication code," but, "for a new protocol design, a ciphersuite with HMAC-MD5 should not be included."

[^3]: Never share your secret. Obviously.
