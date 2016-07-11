---
layout: post
title: Matching known_hosts entries with SSHFP (manually!)
category: admin
---
Have you ever encountered following message:
{% highlight text %}
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
@    WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!     @
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
IT IS POSSIBLE THAT SOMEONE IS DOING SOMETHING NASTY!
Someone could be eavesdropping on you right now (man-in-the-middle attack)!
It is also possible that a host key has just been changed.
The fingerprint for the ED25519 key sent by the remote host is
SHA256:Nhr7GTYHeMQbqrnDBiDXKK2r8/ZcxxsRO4d9LJScStU.
Please contact your system administrator.
Update the SSHFP RR in DNS with the new host key to get rid of this message.
{% endhighlight %}

Quick lookup: `bash $ dig my.domain.pl ` returns:

{%  highlight text %}
;; ANSWER SECTION:
my.domain.pl.          1853    IN      SSHFP   3 2 9CBDBB1DE1D25563386E108190244F50A41F26D0E49103101348824B 68BCCA26
{% endhighlight %}

and we actually don't know much more. What does it mean? While DNS fingerprint looks like sha256 hash is expected to, one
returned by SSH seems strange(or strangely familiar?). It's base64, isn't it?

{%  highlight bash %}
$ echo "Nhr7GTYHeMQbqrnDBiDXKK2r8/ZcxxsRO4d9LJScStU" | base64 -d
6�6x���� �(����\�;�},��J�base64: invalid input
{% endhighlight %}

Unfortunately it doesn't appear to work. Few google searches and it's clear that it should.
Let's try from the other end:
{%  highlight bash %}
$ echo '9CBDBB1DE1D25563386E108190244F50A41F26D0E49103101348824B68BCCA26' | xxd -r -pn | base64
nL27HeHSVWM4bhCBkCRPUKQfJtDkkQMQE0iCS2i8yiY=
{% endhighlight %}
It worked(why wouldn't it) and it's worth noticing that encoded string is properly padded.  As base64 uses 3 bytes
sequences for encoding, it's obvious that we miss one character in input string(which is sha256 of length 32). Lack of
padding was a problem that errored out previously. In fact it was properly decoded, but the error at the end was
misleading. Appending single equal sign resolves the issue(as well as discarding error from base64):

{%  highlight bash %}
$ echo "Nhr7GTYHeMQbqrnDBiDXKK2r8/ZcxxsRO4d9LJScStU=" | base64 -d | xxd -pn -c32 -l32
361afb19360778c41baab9c30620d728adabf3f65cc71b113b877d2c949c4ad5
{% endhighlight %}

Unfortunately fingerprints still does not match. Carefully analyzing dig output(with <a
href="http://unix.stackexchange.com/a/121881" rel="nofollow">this SO</a> as a guide) reveals that available ssh record
is for ECDSA key and the one being used is ED25519. In my case this is effect of recently done SSH hardening.

Knowing how ssh fingerprints are represented allows creating it by hand(the key is already in known_hosts, thus
it's not a MitM, just missed configuration step):

{%  highlight bash %}
$ echo "SSHFP 4 2" $(echo "Nhr7GTYHeMQbqrnDBiDXKK2r8/ZcxxsRO4d9LJScStU=" | base64 -d | xxd -l32 -u -c32 -pn)
SSHFP 4 2 361AFB19360778C41BAAB9C30620D728ADABF3F65CC71B113B877D2C949C4AD5
{% endhighlight %}

After logging to the server it can also be created using:
{%  highlight bash %}
$ ssh-keygen -r my.domain.pl -f /etc/ssh/ssh_host_ed25519_key.pub
my.domain.pl IN SSHFP 4 2 361AFB19360778C41BAAB9C30620D728ADABF3F65CC71B113B877D2C949C4AD5
{% endhighlight %}

As a side note, it used to be much simpler, as md5 sums have been used in simple hex format. Change to sha256
fingerprints in OpenSSH 6.8 introduced base64 for displaying them. For some reason it also introduced that misleading <a
href="https://anongit.mindrot.org/openssh.git/tree/sshkey.c#n946" rel="nofollow">padding trimming</a>.
