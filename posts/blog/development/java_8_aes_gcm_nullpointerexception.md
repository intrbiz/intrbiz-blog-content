Date: 2014-12-09
Author: Chris Ellis
Category: blog/development
Code: java
---
# Java 8, AES GCM and the dreaded NullPointerException

A component of my monitoring system is a non-blocking HTTP and HTTPS check 
engine, this simply makes a HTTP or HTTPS connection to a host and checks the 
response.

I was hitting an odd bug in Java 8 crypto provider, when connecting to TLS 1.2 
servers offering AES GCM I was seeing a `NullPointerException` being thrown.  On 
some servers this fault would be intermittent, however on one particular server, 
the exception is consistent and reliably repeatable.

The exception is being thrown deep from within the bowels of the Sun crypto 
provider:

    Caused by: java.lang.NullPointerException
        at java.lang.System.arraycopy(Native Method)
        at com.sun.crypto.provider.GCTR.reset(GCTR.java:125)
        at com.sun.crypto.provider.GCTR.doFinal(GCTR.java:116)
        at com.sun.crypto.provider.GaloisCounterMode.doLastBlock(GaloisCounterMode.java:343)
        at com.sun.crypto.provider.GaloisCounterMode.decryptFinal(GaloisCounterMode.java:511)
        at com.sun.crypto.provider.CipherCore.finalNoPadding(CipherCore.java:1023)
        at com.sun.crypto.provider.CipherCore.doFinal(CipherCore.java:960)
        at com.sun.crypto.provider.AESCipher.engineDoFinal(AESCipher.java:479)
        at javax.crypto.CipherSpi.bufferCrypt(CipherSpi.java:830)
        at javax.crypto.CipherSpi.engineDoFinal(CipherSpi.java:730)
        at javax.crypto.Cipher.doFinal(Cipher.java:2416)
        at sun.security.ssl.CipherBox.decrypt(CipherBox.java:535)
        at sun.security.ssl.EngineInputRecord.decrypt(EngineInputRecord.java:216)
        at sun.security.ssl.SSLEngineImpl.readRecord(SSLEngineImpl.java:968)
        at sun.security.ssl.SSLEngineImpl.readNetRecord(SSLEngineImpl.java:901)
        at sun.security.ssl.SSLEngineImpl.unwrap(SSLEngineImpl.java:775)
        at javax.net.ssl.SSLEngine.unwrap(SSLEngine.java:624)
        at io.netty.handler.ssl.SslHandler.unwrap(SslHandler.java:883)
        at io.netty.handler.ssl.SslHandler.unwrap(SslHandler.java:828)
        at io.netty.handler.ssl.SslHandler.decode(SslHandler.java:803)
        at io.netty.handler.codec.ByteToMessageDecoder.callDecode(ByteToMessageDecoder.java:226)

Having previously played around compiling the OpenJDK 8, I had a copy of the 
OpenJDK 8 source tree laying around.  The `reset()` method of the 
`com.sun.crypto.provider.GCTR` class is somewhat simple:

    /**
     * Resets the content of this object to when it's first constructed.
     */
    void reset() {
        System.arraycopy(icb, 0, counter, 0, icb.length);
        counterSave = null;
    }

Given the above code, it is most likely that either or the `counter` or `icb` 
member variables are null.  Given that the `icb` member variable is declared 
`final` and is initialised in the constructor and that the `icb` variable is 
used within the constructor it is most likely that the `counter` member variable 
is `null` leading to this exception.

By further reading through the `GCTR` class.  We can see that the only way the 
`counter` member variable can be nullified, is by calling the `reset()` method 
followed by the `restore()` method.

The obvious fix for this bug is to patch `reset()` to be:

    void reset() {
        this.counter = this.icb.clone();
        counterSave = null;
    }

This avoids the possibility of a `NullPointerException` by avoiding copying 
into a `null` `counter` member variable.

I patched the `GCTR` class with the above code, sprinkled in a few logging 
statement so as to trace relevant method calls and workout the sequence of 
events.  I then compiled the OpenJDK and executed my test case with it.

Sadly rather than this being the end of the story, it turned out to be the 
middle.  The original `NullPointerException` had now morphed into another 
`NullPointerException` coming from a different part of the `GCTR` class.

Looking at the sequence of events, the internal state is getting inconsistent, 
by a call to `reset()` followed by a call to `restore()`.

After a few more cycles of adding logging, compiling and running my tests.  I've 
tracked the cause down to the method `doFinal()` in the class 
`com.sun.crypto.provider.CipherCore`.  This method calls `save()` and 
`restore()` in order to handle the restoring of state in the event the output 
buffer is to small.  However `reset()` can be called by the processing on the 
input buffer by the `GCTR` class, by a call to `reset()` from the `doFinal()`.

So in the event the output buffer is smaller than the plain text, the magic 
sequence of `reset()` followed by `restore()` will be called.  This would align 
with the intermittent nature of this bug I've been seeing.

Having patched the `restore()` method to the following:

    /**
     * Restores the content of this object to the previous saved one.
     */
    void restore() {
        if (this.counterSave != null)
            this.counter = this.counterSave;
    }

I see no issues and AES/GCM works fine connecting to all hosts in my test suite.

When you think it through, it seems acceptable and within the definition of the 
implicit contract that, calling `restore()` when no state is saved, it should 
be a no-op.

Now to hope OpenJDK / Oracle include my somewhat minor patches into Java 8 to 
address this bug :)
