+++
title = "Bachelor Degree Project: Password Generation using ambient sound"
date = 2026-01-25T19:31:10+01:00
draft = false
+++
My bachelor degree project that investigates if using regular ambient sound can be used to generate secure passwords. 

In short: it is not great. Using two computers to record audio simultaneously resulted in one computer showing terrible entropy according to the NIST 800-90B test suite, while the other showed great entropy. When using the recorded audio as an entropy source for our underlying hash-based RNG, our RNG passed the NIST 800-22 tests. Passwords were not similar according to our amateur testing. We would advice against using this method to generate passwords.

[Read it online](https://www.diva-portal.org/smash/get/diva2:1989806/FULLTEXT01.pdf)

# Harnessing Ambient Sound for High-Entropy Password Generation
## Abstract
> In the digital age of today passwords is the primary way of keeping your online
accounts safe. One common way to generate and keep track of passwords are
password managers. These managers often include pseudorandom password
generators which are not truly random. A true random password generator is
hosted by Random.org that offers true randomness as a service. The service
utilizes the randomness of atmospheric noise as a source of entropy. This
project looked at the feasibility of using the least significant bits found in
ambient sound recordings as a source of entropy for password generation.
A comparison was done with Random.org. Testing was done using the
NIST SP 800-90B and NIST SP 800-22 test suite to estimate entropy and
randomness. Adversarial resistance was tested by recording simultaneously
and measuring the mutual information between generated passwords and
collected least significant bits. The results revealed that the highest estimated
percentage of the theoretical maximum entropy was 99.5%, slightly below
Random.orgs lowest maximum entropy percentage (99.8%), and the lowest
did not pass the 800-90B tests and was not awarded an estimate. To pass
the NIST 800-22 test suite comparably to Random.org, a minimum of 64 bits
per randomly generated integer were needed to be collected for a hash-based
RNG. Per 16 bit audio sample, only extracting the least significant bit produced
results comparable to Random.org in all recording scenarios. Produced user
and adversary passwords had a minimum hamming distance of 7. Using bit
strings of 640 concatenated least significant bits to pick 10 random symbols
resulted in a minimum hamming distance of 266 between user and adversary
bitstrings. Tests did not find any significant mutual information using the
discovered optimal settings.