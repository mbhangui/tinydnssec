# DNSSEC for tinydns

This project adds DNSSEC support to D. J. Bernstein's [tinydns](http://cr.yp.to/djbdns.html).

It consists of two parts (mostly):

* tinydns-sign, a perl script for augmenting a tinydns-data file with DNSSEC-related RRs, and
* a patch to tinydns / axfrdns to make them produce DNSSEC-authenticated answers.

The patch tries to preserve the behaviour of tinydns/axfrdns wrt non-DNSSEC queries, with these noteworthy exceptions:

* The interpretation of wildcard records now matches the description in RFC-1034 section 4.3.3. Specifically, if there's a wildcard *.x and a record for a.x, then a query for y.a.x will *not* be answered using the wildcard (for a label 'a' and series of labels 'x' and 'y'). This change is required for signed domains, because authentication of negative responses requires a common understanding between client and server about the meaning of wildcards.
* EDNS0 in queries will be honoured also for non-DNSSEC queries, i.e.  tinydns may produce answers exceeding 512 bytes. (There is a hard limit of 4000 bytes, though.) This *can* lead to problems on IPv6 networks.
* TXT records are split into character-strings of 255 bytes, not 127. This is not really a DNSSEC-related change, but this is kind of a FAQ [5] and tinydns-data and tinydns-sign must agree on how this is handled or the generated RRSIG won't match.
* The patch includes a fix for the broken CNAME handling in tinydns. See [6] for a description of the problem. The patch referenced by that description conflicts with fefe's IPv6 patch and requires further modifications for DNSSEC, so I decided to roll my own solution.

Be careful with publishing signed zones as a secondary nameserver: the modified tinydns/axfrdns require certain helper RRs in the database to simplify locating NSEC3 records. Without these helpers, tinydns cannot generate valid negative response nor valid wildcard responses.

Axfrdns *will* publish these helper RRs, other primaries will most likely *not*.

## HOWTO

0. Install tinydns-sign and patched tinydns/axfrdns.

1. Generate key(s). See the tinydns-sign manpage for details.

   It is common practice to have a "Key signing key" (KSK, with flags=257)
   and a "Zone signing key" (ZSK, with flags=256). The KSK is used only for
   signing the DNSKEY RRs, the ZSK is used for signing the rest. The KSK is
   more difficult to change because it is used in the delegating domain's
   referral, therefore it usually has more bits. The ZSK is used for signing
   all the other records, and is therefore usually shorter and changed more
   frequently.

   You should keep the keys in a safe place (outside the tinydns ROOT), e. g.
   in a directory "keys" located above the ROOT.

2. Add the K pseudo records from the key files to your tinydns-data file.
   Also, add a P pseudo record for each signed zone.

3. Adapt the Makefile to pipe your data file through tinydns-sign before
   before running tinydns-data, e. g.

   ```
   data.cdb: data update
     tinydns-sign ../keys/* <data >data.tmp
     mv data.tmp data
     tinydns-data
     rm -f update
   ```

   ```
   update:
     touch update
   ```

4. Run make.

5. Set up a cronjob to periodically re-sign your data file before the
   signatures expire.

6. TEST! For example:

   * Use dig axfr <domain> @<server> and validate the result with a dnssec zone validator, like yazvs [1].
   * Use an online DNS or DNSSEC test tool. See [2] for a list.

7. Read RFC-4641 [3] to get a feeling for what is explicitly not called
   "Best Current Practices". :-)

   In particular, think about key lifetime and how to do a key rollover.

8. Sacrifice a few small animals to a deity of your choice. Get yourself a
   drink for really tough guys, like prune juice [4].

9. If you feel brave, contact your upstream delegator to publish DS records
   for your zone.

   Note that this is a really good way to cut yourself off from the rest of the
   internet. You've been warned, so don't blame me.

## What is djbdns and why does it need IPv6?

Most people agree that IPv6 will come sooner or later, but obviously you need a DNS infrastructure that supports IPv6. djbdns is a full blown DNS server which outperforms BIND in nearly all respects. However, it does not support IPv6 out of the box.

Fortunately, Dan Bernstein (the author of djbdns) has defined a very clean API that made the conversion possible in a few days.

## What does your diff do?

* The current version adds support for AAAA records (those are the DNS records that store IPv6 numbers). tinydns-conf will now create `/etc/tinydns/add-host6` and `/etc/tinydns/add-alias6`, and data can now contain records of type "6" and "3". Also, dnsq now understand AAAA records and a new program called "dnsip6" is the IPv6 equivalent of the old dnsip.
* [new] Automatic internal lookup of some reserved IPv6 addresses (like "::1"). There is also experimental IPv6 transport support.

This diff also integrates an ipv6 port of Russ Nelson's anti-Verisign patch.  Boycott saboteurs!

## What does not work yet?

tinydns-edit won't accept IPv6 addresses for NS or MX records yet. I haven't even started to look at axfr, but it should just work if you use my IPv6 patches for ucspi-tcp.

## How do reverse lookups work?

The reverse lookup for 2001:658:0:2:2e0:18ff:fe98:b03d looks like this:

```
d.3.0.b.8.9.e.f.f.f.8.1.0.e.2.0.2.0.0.0.0.0.0.0.8.5.6.0.1.0.0.2.ip6.int
```

My patch will put a record like this in your data.cdb, but you still need to get a delegation for your range. For a /64, the delegation would mean leaving half the digits away, as in 2.0.0.0.0.0.0.0.8.5.6.0.1.0.0.2.ip6.int. Talk to your ISP about this!

There also is a supposedly new and better scheme for doing DNS for IPv6, and it employs bit strings, DNAME and A6 records, which are really broken by design.  Dan has a good write-up about this issue. Rumour has it that the IETF has seen the light and killed this mindblowingly bad proposal. I have not implemented it and probably will not in the future. It involves the domain ip6.arpa, in case you see that somewhere. (News July 2002: The IEFT really has seen the light)

## Where can I get the patch?

Look at [here](https://www.fefe.de/dns/)

Simply download djbdns-1.05-test28.diff.xz [sig]. Beware: unified diff format (you probably need GNU patch to apply it).

[News!] By request I also host this IXFR patch which makes axfrdns work with BIND9 slaves using IXFR. This patch was originally posted by Luc Pardon, but it is probably easier to find here than in the list archives.

[News!] If you want IPv6 and DNSSEC, you'll find there are patches for both but they conflict. Here is a merged patch, thanks to Henryk.

## What patches have been applied
This dist was created by Henryk Plötz The steps he took, in order:

 1. Import djbdns-1.05.tar.gz. No signature check was made since no signed version
    is available, but I checked that I was using the same package as
    Ubuntu/Debian.
 2. Apply djbdns-1.05-test27.diff.bz2. I checked Fefe’s signature and verified his
    key’s fingerprint using a separate channel.
 3. Apply 0003-djbdns-misformats-some-long-response-packets-patch-a.diff from the
    Ubuntu package.
 4. Apply 0004-dnscache.c-allow-a-maximum-of-20-concurrent-outgoing.diff from the
    Ubuntu package.
 5. Apply djbdns-ipv6-make.patch. No signature check was done, but the patch is
    trivial.
 6. Import tinydnssec-1.05-1.3.tar.bz2. I checked Peter’s signature and verified
    his key through the web of trust.
 7. Apply djbdns-1.05-dnssec.patch from the aforementioned package.
 8. Small fixup for conf-cc and conf-ld: Do not use diet for compilation or
    linking (was introduced with Fefe’s patch).
 9. Small fixup for tinydns-sign.pl: Use Digest::SHA instead of Digest::SHA1.
10. Small fixup for run-tests.sh: GNU tail does not understand the +n syntax.
11. Small fixup for run-tests.sh: Need bash, say so (not all /bin/sh are bash).

## Other patches that have been applied

```
dnscache.c: Read longer buffer over TCP connections
Original author: Frank Denis
http://download.pureftpd.org/misc/dnscache-dos.c
http://www.openwall.com/lists/oss-security/2014/02/17/3 

cache.c: dnscache siphash patch
Original author: Frank Denis
https://00f.net/2012/06/26/dnscache-poisoning-and-siphash/
http://www.openwall.com/lists/oss-security/2014/02/10/4 

tdlookup.c: merged one second patch
Original author: Lennert Buytenhek
http://tinydns.org/one-second.patch 

dns_transmit.c: patch to use TCP sockets for ANY queries.
Original author: Scott Brynen
http://marc.info/?l=djbdns&m=135734230220249&w=2 

query.c: patched to fix the ghost domain vulnerability.
Original author: Peter Conrad
https://github.com/pjps/ndjbdns/commit/c90dbbbac5622e2744733f39e037263e63b51266 

query.c: patched to cache SOA records.
Original author: Jeff King
http://www.your.org/dnscache/0002-dnscache-cache-soa-records.patch 

response.c: Patched with the latest 'response_len < 16384' patch from Matthew
Dempsky.
Original author: Matthew Dempsky
http://marc.info/?l=djbdns&m=123613000920446&w=2 

dns_transmit.c: Patched with the 'd->pos = 0' and 'char udpbuf[4097]' patch
from Matthew Dempsky.
Original author: Matthew Dempsky
http://marc.info/?l=djbdns&m=119983010611174&w=3 
http://marc.info/?l=djbdns&m=122368590802063&w=2 
```

Luc Pardon has instructions to make axfrdns work with BIND9 slaves who use IXFR.
http://www.fefe.de/dns/djbdns-1.05-ixfr.diff.gz

dnsroots.global.patch

# LICENSE

(C) 2012 Peter Conrad <conrad@quisquis.de>

This program is free software: you can redistribute it and/or modify
it under the terms of the GNU General Public License version 3 as
published by the Free Software Foundation.

This program is distributed in the hope that it will be useful,
but WITHOUT ANY WARRANTY; without even the implied warranty of
MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
GNU General Public License for more details.

You should have received a copy of the GNU General Public License
along with this program.  If not, see <http://www.gnu.org/licenses/>.

# References

[1] http://yazvs.verisignlabs.com/ .
[2] http://www.bortzmeyer.org/tests-dns.html
[3] http://tools.ietf.org/html/rfc4641
[4] http://en.memory-alpha.org/wiki/Prune_juice
[5] http://marc.info/?l=djbdns&m=120848817816960&w=2
[6] http://homepage.ntlworld.com/jonathan.deboynepollard/FGA/djbdns-problems.html#tinydns-alias-chain-truncation
