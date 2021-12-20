---
title: "Verifiable Voting: A Primer"
date: 2019-12-05T14:48:30-08:00
draft: false
enableMath: true
---

Given the level of distrust in election systems in recent years, I became curious about verifiable voting systems -- systems in which you can ensure that your vote was _really_ counted, and counted correctly. Systems in which everyone (or at least interested parties) can verify that election results precisely reflect the votes cast. The verifiable voting system I'll describe is pretty close to "regular" voting for voters. They don't need to care the election is e2e verified, they just need to vote like normal. Only people interested in verifying need to do anything beyond casting a standard ballot. This blog post discusses the current work in verifiable voting systems, both in theory and in practice.

Verifiable voting is hard because systems must satisfy two requirements that appear to be at odds:

1. Every individual should be able to verify their vote was properly counted.
2. No individual should be able to prove who they voted for. 

Any system where a 3rd party can determine, with certainty, the vote of a specific individual is vulnerable to coercion.

Most of these systems rely on cryptography -- Rivest created a system which _does not_ rely on cryptography, although the complexity of filling out the ballot seems too funky to use in practice.[^triple]  But, modern cryptographic voting systems have enabled an end-user voting experience essentially identical to what voters expect but with the added bonus of end-to-end verifiability.

In this blog post I'll provide a high level overview of how verifiable voting functions, followed by a slightly more mathy look at how it all works under the hood. There several different verifiable voting systems that have been conceived of -- I'll be describing a system called Prêt à Voter (~~the only one, to my knowledge, that's been used in a real election~~. Someone recently brought Scantegrity to my attention, which was also used in [two US elections.](https://www.chaum.com/publications/Scantegrity-II-Municipal-Election-at-Takoma-Park-the-first-E2E-Binding-Governmental-Election-with-Ballot-Privacy.pdf))[^realelection]. Going into the 2020 Election, Microsoft has been pushing Election Guard, a system based on Homomorphic encryption.[^microsoft] I won't go into that system here.

**Disclaimer**: _I am not an expert in any of these subjects. Please don't build a voting system from my blog post._

## At a High Level

The first key insight in verifiable voting is the "split ballot". The candidate labels of the ballot are on the left side. The right side of ballot is where the actual votes are recorded. Each ballot has a randomized order -- this means that a right-hand ballot alone can't reveal who the vote was cast for. The right side also contains an encrypted QR code or piece of text that securely indicates the ballot ordering so that the vote can be counted. When a voter votes, they mark the appropriate option on the right side, split off and destroy the left-hand side of the ballot, and submit the right-hand side. They'll receive a copy of the right-hand side of their ballot as a receipt. 
![Example Ballot](/images/ballot.svg)

This sort of voting is quite similar to a "normal" polling place experience, but, with the addition of a receipt.

The right-hand side of each ballot is then posted on the internet -- the literature describes this phase as the "Web bulletin board" or WBB. Because the ballots posted are only the right-hand sides, you can't determine what candidate a ballot has been cast for, but, crucially, you can find _your_ vote and make sure the selection is matches your receipt. _Election security depends on a non-trivial fraction of voters validating their vote online._

Next, we need to actually count the ballots in a way that can be verified. The key idea is that the ballots are shuffled in a way that we can be sure that no individual vote has been changed, but we don't know which input vote corresponds to which output vote. Once the ballots come out the other end of the shuffle, we know the actual value of each vote, but _not_ the original vote it came from. This step be cannot easily audited by laypeople, but they can delegate to a trusted auditor.

If you're satisfied by this explanation, feel free to jump to the end. Otherwise, keep reading!

## Under the hood
There's a lot of handwaving in the high level explanation, and a lot of deference to assumption. I'll try to get rid of some of those assumptions here. To understand this section, you'll need a basic understanding of [public-key/asymmetric cryptography](https://en.wikipedia.org/wiki/Public-key_cryptography). In one sentence: Public key crypto means that anyone can encrypt a message that only the person with the secret key can read. You don't need to know the secret key to encrypt a message, only to decrypt it.

### Construction of Ballots
Suppose the election has a secret key `\(K_s\)` and a corresponding public key `\(K_p\)`[^manykeys]. Ballot generation machines each are provided with the public key, allowing them to encrypt data. The voting machine generates a randomly ordered ballot and encrypts the ordering along with a random salt.[^salt] The encrypted ordering is printed onto the right-hand side of the ballot as a QR code (or text, in the case of simple ballots). In the case of more complex elections, eg. ranked choice voting, or jurisdictions that have laws about the order candidates appear on a ballot, voters can input their results on an electronic device in a traditional ballot format. The machine will then generate & print a completed ballot on demand that can be verified and perforated in the same way as the image above.

#### How do you know your ballot was properly constructed?
Proving that ballots are valid is tricky -- one might imagine malicious or buggy voting machines that produced QR codes that didn't match the ordering on the ballot. One solution is to allow ballots to be "spoiled" -- at random, voters or polling place workers may decide to "spoil" a ballot they've been given and request an additional ballot to vote with. Since the ballots are encrypted with a public key, the only thing required to verify a ballot is properly constructed is the random salt. The ballot can be immediately spoiled for the voter by querying the voting system for the corresponding salt. This ballot is then invalidated, and the voter can confirm that the encrypted QR code matches. There are several similar systems described in [Culnane et al, sec 3.1.3](https://arxiv.org/pdf/1404.6822.pdf) that all hinge on the same principle of invalidating a given ballot in order to prove the the encryption was correct.

### Counting the Votes
This is where things start to get interesting -- we need a system where all the known (but encrypted) votes go in, and the decrypted votes come out but without the ability to match a decrypted vote to its corresponding input vote. Prêt à Voter uses a system called "mix nets", originally conceived by David Chaum to support "Untraceable Electronic Mail, Return Addresses, and Digital Pseudonyms."[^mixnetpaper] The idea is that we want to create a "mixer" where many messages go in, many messages go out, and we have no way of knowing (better than guessing), which output corresponds to which input. Crucially, mixnets offer a guarantee that no individual messages have changed, even though the input has been shuffled.

There are several ways to use mixnets to support voting privacy, but the basic concept is as follows: There are `\(n\)` mixnet servers[^server]. The first mixnet server has the private key of the election, `\(K_s\)`.[^mixpk] The subsequent mixnet servers generate their own private/public key pairs. Each mixnet server reveals its public key.

Each layer of the mixnet performs the following operations:

1. Decrypt the incoming votes (the votes were encrypted with the corresponding public key of the layer server).
2. Change the random salt of each vote. Shuffle the ordering of the ballots.
3. Re-encrypt each vote with the public key of the next layer (or in the case of the final mixnet, simply output the decrypted votes).

**Since all the encryption is public key encryption, by selectively revealing salts, the mix servers can prove that they are shuffling ballots without changing their contents.**
Mixnets provide two things:

1. Shuffle & anonymize the input ballots to provide privacy
2. Provide a framework where each group of two servers can prove it did not alter votes to provide robustness.

### Verifying the count
I'm not going to go into depth here. For that, you'll want to read [Making Mix Nets Robust For Electronic Voting By
Randomized Partial Checking by Jakobsson, Jules and Rivest](https://people.csail.mit.edu/rivest/voting/papers/JakobssonJuelsRivest-MakingMixNetsRobustForElectronicVotingByRandomizedPartialChecking.pdf). Rather, I'll attempt to provide a summary that will hopefully satisfy most readers.

The key insight to verifying a mixnet is that we can force each layer to reveal a subset of its connections without violating the privacy of a single ballot. **When a mixnet reveals a connection, it allows an outside observer to verify that a ballot was not tampered with.** Specifically, we'll focus on pairs of servers. For each server, we force it to reveal a subset of a connections, such that no input can be traced end-to-end through the pair of servers. If one server reveals the corresponding input and output, the subsequent server _will not_ be required to reveal the corresponding input and output.

Since the mixnet servers use public key encryption, the only piece of information required to be leaked for verification is the random salt -- with that salt, an exterior observer can use the public key to reproduce the encryption and be assured the that contents of the votes were unchanged.

![Mixnet Randomized Partial Checking](/images/mixnet.svg)

In the figure above, I've used color to denote the identities known about each ballot. Crucially, we don't know how any output layer ballot corresponds to input layer ballots even though we know 50% of the connections in each layer. The [paper](https://people.csail.mit.edu/rivest/voting/papers/JakobssonJuelsRivest-MakingMixNetsRobustForElectronicVotingByRandomizedPartialChecking.pdf) provides the privacy and robustness proof for this scheme -- I won't get into it here.

## Wrap Up
Hopefully this serves as a satisfying summary for how verifiable voting can work in practice. One major caveat of these schemes is that it doesn't seem like people using the system necessarily understand what was going on. More than have of respondents to a poll after the election in Australia thought that someone who had their receipt knew who they voted for. Though this doesn't reduce the security or pose any problems for the election, at the end of the day, systems like this will only be implemented if people really understand the benefits they provided.

If I got something wrong, as always, please let me know! You can make a pull request or file an issue on [Github](https://github.com/rcoh/rcoh-dot-me-v2).
***
{{% subscribe %}}

[^triple]: Rivest, 2006: The system involves 3 identical ballots. Voters vote twice for the candidate they want and once for the candidates that they don't want. The resulting vote totals are offset by the number of total voters, so the original vote totals can be discovered with simple counting. Besides being impractical in practice, it also doesn't fully protect from vote buying. [Full Paper](https://people.csail.mit.edu/rivest/Rivest-TheThreeBallotVotingSystem.pdf)

[^realelection]: Pret a Voter was used in the state of Victoria, Australia which uses Ranked-Choice voting. [Paper describing how Pret a Voter was applied in practice.](https://arxiv.org/pdf/1404.6822.pdf). [Follow up paper describing results](http://epubs.surrey.ac.uk/809386/1/vVote.pdf). Scantegrity was used in two elections in Maryland. Scantegrity is a "front-end" for verifiable voting (like the tear apart ballot in Pret a Voter). It is used in combination with a backend like Mixnets to actually count the votes.

[^microsoft]: [Verifiable Secret Ballot Elections, Josh Daniel Cohen Benaloh, 1996](https://www.microsoft.com/en-us/research/wp-content/uploads/1987/01/thesis.pdf) [SDK Spec](https://github.com/microsoft/ElectionGuard-SDK-Specification) 

[^manykeys]: In reality, elections use [Threshold Encryption](https://en.wikipedia.org/wiki/Threshold_cryptosystem) to separate the private key material between multiple parties such that only a quorum can decrypt the ballots. For simplicity, I'm describing a system with a single public/private key pair.

[^salt]: The salt (or sometimes referred as random padding) is crucial for ballot secrecy -- since there a finite number of permutations of the ballot, without it, an adversary could determine the contents of a vote simply by enumerating possible ballot permutations and matching the resulting cipher texts. (Recall that anyone can produce an encrypted ballot because the ballots are encrypted with a public key).

[^mixnetpaper]: Untraceable electronic mail, return addresses, and digital pseudonyms. Communications of the ACM, 24(2):84–88, 1981. [PDF](https://www.freehaven.net/anonbib/cache/chaum-mix.pdf)

[^server]: These are referred to as servers because they came from email, but there is no real reason why, in the case of voting, that each server need to be separated. It can be an iterative process on one computer.

[^mixpk]: In reality, each mixnet server contains some cryptographic information that, only when combined with other mixnet servers, can decrypt the ballots. See [Threshold Cryptosystem](https://en.wikipedia.org/wiki/Threshold_cryptosystem). For purposes of explanation, I've elided this detail in the main discussion.



