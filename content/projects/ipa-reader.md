---
title: "Wikipedia IPA Reader"
date: 2017-12-30T12:17:50-05:00
short: "[Chrome extension](https://chrome.google.com/webstore/detail/wikipedia-ipa-speaker/jkgihpigffcfeebgedpklldebdibbnne) that allows you to listen to IPA on Wikipedia."
gh_url: https://github.com/rcoh/ipa-reader
draft: false
---
If you click on IPA (international phonetic alphabet) text on wikipedia, it links to a generic IPA page, which I always found frustrating. 
I created a [chrome extension](https://chrome.google.com/webstore/detail/wikipedia-ipa-speaker/jkgihpigffcfeebgedpklldebdibbnne) which adds a play button next to IPA text to read it aloud.

[![ab.dot.png](/images/ipa-example.png)](https://chrome.google.com/webstore/detail/wikipedia-ipa-speaker/jkgihpigffcfeebgedpklldebdibbnne)

Done in an afternoon, it's almost certainly the highest users:time ratio of any project I've done. It simply extracts the IPA text from wikipedia and then uses Amazon Polly, a text-to-speech service provided by AWS to generate sound.



